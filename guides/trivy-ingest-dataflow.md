---
layout: default
title: Trivy Ingest Data Flow
parent: Platform Guides
nav_order: 15
---

# Trivy Ingest Data Flow

Tài liệu này mô tả luồng dữ liệu khi ingest kết quả scan từ Trivy vào hệ thống Exploop, bao gồm cả vulnerabilities và SBOM (Software Bill of Materials).

---

## Tổng quan

Trivy scan tạo ra 2 loại dữ liệu chính:
1. **Vulnerabilities** - Các lỗ hổng bảo mật (CVE) trong dependencies
2. **SBOM (Dependencies)** - Danh sách tất cả packages/components trong project

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TRIVY SCAN INGEST FLOW                            │
└─────────────────────────────────────────────────────────────────────────────┘

  Agent                          API                           Database
    │                             │                               │
    │  1. Trivy JSON Output       │                               │
    │  ─────────────────────►     │                               │
    │  (trivy.Parser)             │                               │
    │                             │                               │
    │  2. Convert to EIS Report   │                               │
    │  ┌─────────────────────┐    │                               │
    │  │ Report {            │    │                               │
    │  │   Findings[]        │────────►  3. POST /ingest          │
    │  │   Dependencies[]    │    │                               │
    │  │   Assets[]          │    │                               │
    │  │ }                   │    │                               │
    │  └─────────────────────┘    │                               │
    │                             │                               │
    │                             │  4. Process Assets            │
    │                             │  ─────────────────────────►   │
    │                             │  (Upsert to assets table)     │
    │                             │                               │
    │                             │  5. Process Findings          │
    │                             │  ─────────────────────────►   │
    │                             │  (Upsert to findings table)   │
    │                             │                               │
    │                             │  6. Process Dependencies      │
    │                             │  ┌─────────────────────┐      │
    │                             │  │ For each dependency:│      │
    │                             │  │  a. Upsert Component│──────► components
    │                             │  │  b. Link to Asset   │──────► asset_components
    │                             │  └─────────────────────┘      │
    │                             │                               │
```

---

## Chi tiết từng bước

### Bước 1: Agent Parse Trivy Output

**File**: `sdk/pkg/scanners/trivy/parser.go`

Agent nhận output JSON từ Trivy và parse thành 2 phần riêng biệt:

```go
// trivy/parser.go
func (p *Parser) Parse(data []byte, opts *core.ParseOptions) (*eis.Report, error) {
    var trivyReport Report
    json.Unmarshal(data, &trivyReport)

    report := &eis.Report{
        Findings:     []eis.Finding{},
        Dependencies: []eis.Dependency{},
    }

    for _, result := range trivyReport.Results {
        // Parse vulnerabilities → Findings
        for _, vuln := range result.Vulnerabilities {
            report.Findings = append(report.Findings, p.parseVulnerability(&result, &vuln))
        }

        // Parse packages → Dependencies (SBOM)
        for _, pkg := range result.Packages {
            report.Dependencies = append(report.Dependencies, p.parsePackage(&result, &pkg))
        }
    }
    return report, nil
}
```

### Bước 2: Convert to EIS Report

**EIS (Exploop Interchange Schema)** là định dạng chuẩn để trao đổi dữ liệu giữa Agent và API.

```go
// eis/report.go
type Report struct {
    Version      string       `json:"version"`
    Metadata     Metadata     `json:"metadata"`
    Tool         Tool         `json:"tool,omitempty"`
    Assets       []Asset      `json:"assets,omitempty"`
    Findings     []Finding    `json:"findings,omitempty"`      // Vulnerabilities
    Dependencies []Dependency `json:"dependencies,omitempty"`  // SBOM
}
```

#### Dependency Structure

```go
// eis/dependency_types.go
type Dependency struct {
    ID           string              `json:"id,omitempty"`
    Name         string              `json:"name"`
    Version      string              `json:"version,omitempty"`
    Type         string              `json:"type,omitempty"`        // library, framework, application
    Ecosystem    string              `json:"ecosystem,omitempty"`   // npm, pypi, maven, composer
    PURL         string              `json:"purl,omitempty"`        // Package URL (unique identifier)
    UID          string              `json:"uid,omitempty"`         // Trivy UID
    Licenses     []string            `json:"licenses,omitempty"`
    Relationship string              `json:"relationship,omitempty"` // direct, indirect
    Path         string              `json:"path,omitempty"`        // manifest file path
    Locations    []DependencyLocation `json:"locations,omitempty"`  // multiple locations
}

type DependencyLocation struct {
    Path        string `json:"path,omitempty"`
    StartLine   int    `json:"start_line,omitempty"`
    EndLine     int    `json:"end_line,omitempty"`
}
```

### Bước 3-5: API Ingest Service

**File**: `api/internal/app/ingest/service.go`

```go
func (s *Service) IngestReport(ctx context.Context, tenantID shared.ID, report *eis.Report) (*Output, error) {
    output := &Output{}

    // Step 1: Process Assets
    assetMap, _ := s.processAssets(ctx, tenantID, report, output)

    // Step 2a: Process Findings (Vulnerabilities)
    s.findingProcessor.ProcessBatch(ctx, tenantID, report, assetMap, output)

    // Step 2b: Process Dependencies (SBOM)
    if len(report.Dependencies) > 0 {
        s.componentProcessor.ProcessBatch(ctx, tenantID, report, assetMap, output)
    }

    return output, nil
}
```

### Bước 6: Component Processor

**File**: `api/internal/app/ingest/processor_components.go`

```go
func (p *ComponentProcessor) ProcessBatch(
    ctx context.Context,
    tenantID shared.ID,
    report *eis.Report,
    assetMap map[string]shared.ID,
    output *Output,
) error {
    for _, dep := range report.Dependencies {
        // a. Tạo/Update Component (global, unique by PURL)
        comp := component.NewComponent(dep.PURL, dep.Name, dep.Version, ecosystem)
        compID, isNew := p.repo.Upsert(ctx, comp)

        if isNew {
            output.ComponentsCreated++
        } else {
            output.ComponentsUpdated++
        }

        // b. Link Component với Asset
        assetDep := component.NewAssetDependency(
            tenantID, assetID, compID,
            dep.Path,           // manifest file path
            dependencyType,     // direct/indirect
        )
        p.repo.LinkAsset(ctx, assetDep)
        output.DependenciesLinked++
    }
    return nil
}
```

---

## Dependencies vs Components: Data Model Mapping

Hệ thống sử dụng hai thuật ngữ khác nhau cho cùng một concept, mỗi thuật ngữ phù hợp với context của nó:

### Terminology Mapping

| Context | Term | Ý nghĩa |
|---------|------|---------|
| **EIS Schema (Input)** | `dependencies[]` | Packages mà project **phụ thuộc vào** |
| **Database (Storage)** | `components` | Packages **tồn tại trong hệ thống** (global catalog) |
| **Junction Table** | `asset_components` | Links giữa assets và components |

### Tại sao dùng 2 terms khác nhau?

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA FLOW                                     │
└─────────────────────────────────────────────────────────────────────┘

  Scanner Output          EIS Schema              Database
  ─────────────          ───────────              ────────

  Trivy packages  ──►  dependencies[]  ──►  components (global)
                           │                      │
                           │                      │
                           ▼                      ▼
                    "What this project     "What packages exist
                     depends on"            in our system"
                    (relationship)          (entity)
```

**1. `dependencies` (EIS/Input)** - Mô tả **quan hệ phụ thuộc**:
- "Project A **phụ thuộc vào** package X"
- Có context: direct/indirect relationship
- Per-asset scope

**2. `components` (Database/Storage)** - Mô tả **thực thể package**:
- "Package X **tồn tại** trong hệ thống"
- Global, unique by PURL
- Theo chuẩn SBOM (CycloneDX, SPDX)

### Industry Standards Comparison

| Tool/Standard | Input Term | Storage Term |
|---------------|------------|--------------|
| CycloneDX | - | `components` |
| SPDX | - | `packages` |
| Trivy | `packages` | - |
| Snyk | `dependencies` | - |
| GitHub Dependabot | `dependencies` | - |
| **Exploop** | `dependencies` | `components` |

### Design Benefits

1. **Deduplication**: `guzzlehttp/guzzle@7.3.0` chỉ lưu 1 lần trong `components` dù 100 assets sử dụng
2. **Global Vulnerability Tracking**: Update CVE một lần, apply cho tất cả assets
3. **Relationship Preservation**: `direct/indirect` được lưu trong junction table `asset_components`
4. **Standards Compliance**: Database schema theo chuẩn SBOM

---

## Database Schema

### Quan hệ giữa các bảng

```
┌─────────────────┐       ┌─────────────────────┐       ┌─────────────────┐
│    components   │       │  asset_components   │       │     assets      │
├─────────────────┤       ├─────────────────────┤       ├─────────────────┤
│ id (PK)         │◄──────│ component_id (FK)   │       │ id (PK)         │
│ purl (UNIQUE)   │  │    │ asset_id (FK)       │──────►│ name            │
│ name            │  │    │ tenant_id           │       │ repository_url  │
│ version         │  │    │ path                │       │ ...             │
│ ecosystem       │  │    │ dependency_type     │       └─────────────────┘
│ vuln_count      │  │    │ manifest_file       │              │
└─────────────────┘  │    └─────────────────────┘              │
        ▲            │                                         │
        │            │                                         │
        │            │    ┌─────────────────┐                  │
        │            │    │    findings     │                  │
        │            │    ├─────────────────┤                  │
        └────────────┼────│ component_id    │  (FK to components)
                     │    │ asset_id        │──────────────────┘
                     │    │ cve_id          │
                     │    │ severity        │
                     │    │ ...             │
                     │    └─────────────────┘
                     │
         (SBOM link) │ (Vulnerability link)
```

**Hai loại relationships:**
1. **SBOM Link** (`asset_components`): Asset sử dụng component nào
2. **Vulnerability Link** (`findings.component_id`): CVE ảnh hưởng component nào

### Bảng `components`

Lưu trữ thông tin global về mỗi package (unique by PURL).

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| purl | VARCHAR | Package URL (unique) |
| name | VARCHAR | Package name |
| version | VARCHAR | Package version |
| ecosystem | VARCHAR | npm, pypi, maven, composer, etc. |
| description | TEXT | Package description |
| homepage | VARCHAR | Package homepage URL |
| vulnerability_count | INT | Number of known vulnerabilities |
| metadata | JSONB | Additional metadata |

### Bảng `asset_components`

Junction table linking assets to components.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| tenant_id | UUID | Tenant isolation |
| asset_id | UUID | Foreign key to assets |
| component_id | UUID | Foreign key to components |
| path | VARCHAR | Manifest file path (e.g., `composer.lock`) |
| dependency_type | VARCHAR | `direct` or `indirect` |
| manifest_file | VARCHAR | Source file name |

**Unique Constraint**: `(asset_id, component_id, path)`

---

## Query Examples

### Asset đang sử dụng components nào?

```sql
SELECT
    c.name,
    c.version,
    c.ecosystem,
    ac.dependency_type,
    ac.path as manifest_file
FROM asset_components ac
JOIN components c ON ac.component_id = c.id
WHERE ac.asset_id = '<asset_id>'
ORDER BY c.name;
```

### Component có vulnerabilities gì?

```sql
SELECT
    f.rule_id as cve_id,
    f.severity,
    f.message as title,
    f.status
FROM findings f
JOIN components c ON f.component_id = c.id
WHERE c.purl = 'pkg:composer/guzzlehttp/guzzle@7.3.0'
ORDER BY
    CASE f.severity
        WHEN 'critical' THEN 1
        WHEN 'high' THEN 2
        WHEN 'medium' THEN 3
        WHEN 'low' THEN 4
    END;
```

### Asset có vulnerabilities nào qua components?

```sql
SELECT
    a.name as asset_name,
    c.name as component_name,
    c.version as component_version,
    f.rule_id as cve_id,
    f.severity,
    f.message as title
FROM assets a
JOIN asset_components ac ON a.id = ac.asset_id
JOIN components c ON ac.component_id = c.id
JOIN findings f ON f.asset_id = a.id AND f.component_id = c.id
WHERE a.id = '<asset_id>'
ORDER BY
    CASE f.severity
        WHEN 'critical' THEN 1
        WHEN 'high' THEN 2
        WHEN 'medium' THEN 3
        WHEN 'low' THEN 4
    END;
```

### Tìm assets bị ảnh hưởng bởi CVE cụ thể

```sql
SELECT DISTINCT
    a.id,
    a.name,
    a.repository_url,
    c.name as vulnerable_component,
    c.version
FROM findings f
JOIN assets a ON f.asset_id = a.id
JOIN components c ON f.component_id = c.id
JOIN asset_components ac ON ac.asset_id = a.id AND ac.component_id = c.id
WHERE f.rule_id = 'CVE-2024-1234'
  AND f.tenant_id = '<tenant_id>';
```

### Thống kê components theo ecosystem

```sql
SELECT
    c.ecosystem,
    COUNT(DISTINCT c.id) as total_components,
    COUNT(DISTINCT ac.asset_id) as assets_using,
    SUM(c.vulnerability_count) as total_vulns
FROM components c
LEFT JOIN asset_components ac ON c.id = ac.component_id
GROUP BY c.ecosystem
ORDER BY total_components DESC;
```

---

## PURL Format

**PURL (Package URL)** là định dạng chuẩn để identify packages:

```
pkg:<ecosystem>/<namespace>/<name>@<version>
```

### Examples

| Ecosystem | PURL Example |
|-----------|--------------|
| npm | `pkg:npm/%40angular/core@12.0.0` |
| pypi | `pkg:pypi/requests@2.28.0` |
| maven | `pkg:maven/org.springframework/spring-core@5.3.20` |
| composer | `pkg:composer/guzzlehttp/guzzle@7.3.0` |
| golang | `pkg:golang/github.com/gin-gonic/gin@1.8.1` |
| cargo | `pkg:cargo/serde@1.0.136` |

---

## Trivy Output Example

```json
{
  "SchemaVersion": 2,
  "ArtifactName": "vapi",
  "ArtifactType": "filesystem",
  "Results": [
    {
      "Target": "composer.lock",
      "Class": "lang-pkgs",
      "Type": "composer",
      "Packages": [
        {
          "ID": "guzzlehttp/guzzle@7.3.0",
          "Name": "guzzlehttp/guzzle",
          "Version": "7.3.0",
          "Identifier": {
            "PURL": "pkg:composer/guzzlehttp/guzzle@7.3.0",
            "UID": "abc123"
          },
          "Locations": [
            { "StartLine": 100, "EndLine": 150 }
          ],
          "Licenses": ["MIT"]
        }
      ],
      "Vulnerabilities": [
        {
          "VulnerabilityID": "CVE-2022-29248",
          "PkgID": "guzzlehttp/guzzle@7.3.0",
          "PkgName": "guzzlehttp/guzzle",
          "InstalledVersion": "7.3.0",
          "FixedVersion": "7.4.5",
          "Severity": "HIGH",
          "Title": "Cross-domain cookie leakage",
          "Description": "Guzzle is a PHP HTTP client..."
        }
      ]
    }
  ]
}
```

---

## API Response

Sau khi ingest thành công, API trả về:

```json
{
  "scan_id": "scan-abc123",
  "assets_created": 1,
  "assets_updated": 0,
  "findings_created": 5,
  "findings_updated": 15,
  "findings_skipped": 0,
  "components_created": 72,
  "components_updated": 0,
  "dependencies_linked": 72,
  "errors": []
}
```

---

## Related Documentation

- [Finding Ingestion Workflow](finding-ingestion-workflow.md) - Chi tiết về finding processing
- [EIS Report Schema](../schemas/eis-report.md) - Schema reference
- [SDK Quick Start](sdk-quick-start.md) - Hướng dẫn sử dụng SDK
- [Ingest API Reference](../api/ingest-api.md) - API endpoints
