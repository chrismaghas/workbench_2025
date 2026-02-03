reference
Position,Team,Grade,Description
Software Developer,Engineering,Level 3,Develops and maintains software applications using modern programming languages
Senior Software Developer,Engineering,Level 4,Leads development projects and mentors junior developers
Product Manager,Product,Level 4,Manages product roadmap and coordinates cross-functional teams
Data Analyst,Analytics,Level 3,Builds analytical models and analyzes large datasets
Senior Data Analyst,Analytics,Level 4,Leads data analytics initiatives and provides strategic insights
User Experience Designer,Design,Level 3,Creates user interfaces and conducts user research
Senior User Experience Designer,Design,Level 4,Leads design strategy and manages design systems
Cloud Engineer,Engineering,Level 3,Manages cloud infrastructure and CI/CD pipelines
Senior Cloud Engineer,Engineering,Level 4,Architects cloud solutions and automation strategies
Marketing Manager,Marketing,Level 4,Develops marketing campaigns and manages brand positioning
Business Analyst,Operations,Level 3,Analyzes business processes and requirements
Senior Business Analyst,Operations,Level 4,Leads business analysis initiatives and stakeholder management
Quality Assurance Engineer,Engineering,Level 3,Designs and executes test plans for software quality assurance
Senior Quality Assurance Engineer,Engineering,Level 4,Leads QA strategy and test automation frameworks
Sales Manager,Sales,Level 4,Manages sales team and develops client relationships
Account Manager,Sales,Level 3,Manages key accounts and drives revenue growth


target
Job Title,Department,Level,Description
Software Engineer,Engineering,L3,Develops and maintains software applications using modern programming languages
Senior Software Engineer,Engineering,L4,Leads development projects and mentors junior engineers
Product Manager,Product,L4,Manages product roadmap and coordinates cross-functional teams
Data Scientist,Analytics,L3,Builds machine learning models and analyzes large datasets
Senior Data Scientist,Analytics,L4,Leads data science initiatives and provides strategic insights
UX Designer,Design,L3,Creates user interfaces and conducts user research
Senior UX Designer,Design,L4,Leads design strategy and manages design systems
DevOps Engineer,Engineering,L3,Manages cloud infrastructure and CI/CD pipelines
Senior DevOps Engineer,Engineering,L4,Architects cloud solutions and automation strategies
Marketing Manager,Marketing,L4,Develops marketing campaigns and manages brand positioning
Business Analyst,Operations,L3,Analyzes business processes and requirements
Senior Business Analyst,Operations,L4,Leads business analysis initiatives and stakeholder management
QA Engineer,Engineering,L3,Designs and executes test plans for software quality assurance
Senior QA Engineer,Engineering,L4,Leads QA strategy and test automation frameworks
Sales Manager,Sales,L4,Manages sales team and develops client relationships
Account Executive,Sales,L3,Manages key accounts and drives revenue growth





# Sample CSV Files for C2C Workflow Testing

This directory contains sample CSV files for testing the C2C (Census-to-Census) workflow upload endpoint.

## Files

### `sample_target_file.csv`
- **Purpose**: Sample target file for C2C workflow
- **Columns**: 
  - `Job Title` (required for mapping)
  - `Department` (optional for mapping)
  - `Level` (optional for mapping)
  - `Description` (optional for mapping)

### `sample_reference_file.csv`
- **Purpose**: Sample reference file for C2C workflow
- **Columns**:
  - `Position` (required for mapping - equivalent to Job Title)
  - `Team` (optional for mapping - equivalent to Department)
  - `Grade` (optional for mapping - equivalent to Level)
  - `Description` (optional for mapping)

## Usage

### 1. Upload Files
Use these files with the upload endpoint:

**POST** `/api/v2/c2c/{project_id}/files`

**Request (multipart/form-data)**:
- `target_file`: `sample_target_file.csv`
- `reference_file`: `sample_reference_file.csv`

### 2. Configure Target File Mapping

**PUT** `/api/v2/c2c/{project_id}/target-file`

```json
{
  "title_colname": "Job Title",
  "department_colname": "Department",
  "level_colname": "Level",
  "function_colname": null,
  "description_colname": "Description"
}
```

### 3. Configure Reference File Mapping

**PUT** `/api/v2/c2c/{project_id}/reference-file`

```json
{
  "title_colname": "Position",
  "department_colname": "Team",
  "level_colname": "Grade",
  "function_colname": null,
  "description_colname": "Description"
}
```

## Data Notes

- **Target file** uses standard column names: `Job Title`, `Department`, `Level`
- **Reference file** uses alternative column names: `Position`, `Team`, `Grade`
- Both files contain 16 sample job records with matching roles (e.g., "Software Engineer" in target matches "Software Developer" in reference)
- Level formats differ: target uses `L3`, `L4` while reference uses `Level 3`, `Level 4`
- This demonstrates the need for level mapping in the workflow

## Level Mapping Example

After uploading and mapping columns, you'll need to create level mappings:

**POST** `/api/v2/c2c/{project_id}/level-mappings`

```json
{ "target_level": "L3", "reference_level": "Level 3" }
```

```json
{ "target_level": "L4", "reference_level": "Level 4" }
```

## Testing Scenarios

These sample files support testing:
1. ✅ File upload (multipart/form-data)
2. ✅ Column mapping with different column names
3. ✅ Level harmonization (L3 → Level 3, L4 → Level 4)
4. ✅ Matching jobs (similar titles should match)
5. ✅ Match alternatives (vector similarity search)
6. ✅ Results download (CSV export)

---

*Last Updated: 2026-02-02*

