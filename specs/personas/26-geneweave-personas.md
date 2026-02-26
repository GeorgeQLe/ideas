# GeneWeave — Persona Subspecs

> Parent spec: [`specs/26-geneweave.md`](../26-geneweave.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Provide a simplified pipeline builder with a curated set of pre-configured "recipe" workflows (e.g., "RNA-seq Differential Expression", "Variant Calling from WES")
- Show contextual explanations for each pipeline module describing what the tool does, why it matters, and what parameters to adjust
- Display a step-by-step execution timeline with plain-language status messages instead of raw log output

**Feature Gating:**
- Allow running pre-built pipelines on small datasets (up to 10 FASTQ files, 50 GB total storage)
- Hide custom module creation, API access, and HIPAA-compliant infrastructure options
- Limit compute to shared-tier resources with queue-based scheduling (no priority compute)

**Pricing Tier:** Free tier (academic email required)

**Onboarding Flow:**
- First-run tutorial processes a public RNA-seq dataset (2 samples) through a pre-built differential expression pipeline in under 15 minutes
- Offer a guided walkthrough explaining each QC metric (Phred scores, adapter content, duplication rates) with visual examples
- Suggest a learning path of 5 progressively complex pipeline exercises using public datasets

**Key Workflows:**
- Select a pre-built pipeline recipe, upload FASTQ files, and launch analysis with default parameters
- Review quality control reports (FastQC, MultiQC) and learn to interpret key metrics
- Run differential expression analysis and explore results through interactive volcano plots and heatmaps
- Export results (count matrices, variant lists, figures) for a course assignment or thesis chapter

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show the full visual pipeline builder with drag-and-drop modules and data type validation between connections
- Provide a "Pipeline Health" panel monitoring execution status, resource usage, and estimated completion time per stage
- Display parameter configuration panels with recommended defaults and brief explanations for each setting

**Feature Gating:**
- Unlock custom pipeline construction from available modules, parametric configuration, and up to 500 GB storage
- Expose batch sample processing and basic pipeline versioning with automatic snapshots
- Hide HIPAA compliance features, custom container imports, and multi-tenant administration

**Pricing Tier:** Pro tier ($59/month)

**Onboarding Flow:**
- Walk through building a custom pipeline by connecting modules (trimming, alignment, variant calling, annotation) on the canvas
- Demonstrate batch processing by uploading multiple samples and launching parallel execution
- Show how to version a pipeline and compare results between parameter configurations

**Key Workflows:**
- Build custom analysis pipelines by connecting pre-built modules with configurable parameters
- Process batch samples (10-50) through a pipeline with parallel execution and progress monitoring
- Modify pipeline parameters and compare results across configurations to optimize analysis quality
- Version pipelines and maintain a history of configurations for reproducibility
- Share pipeline designs and results with lab members for review and discussion

---

### Senior/Expert

**Modified UI/UX:**
- Provide a code-alongside-canvas view showing the generated Nextflow/WDL code for the visual pipeline with direct editing capability
- Expose advanced compute configuration: instance types, spot/on-demand mix, storage tiers, and cost optimization settings
- Support custom Docker container import for adding proprietary or bleeding-edge tools to the pipeline

**Feature Gating:**
- Full access: unlimited storage, priority compute, custom containers, API access, HIPAA/GDPR compliance, and multi-tenant administration
- Expose pipeline-as-code with Git integration for versioning and CI/CD-driven execution
- Enable integration with BaseSpace, S3, GCS, and institutional HPC via hybrid execution

**Pricing Tier:** Enterprise tier ($299/month or institutional contract)

**Onboarding Flow:**
- Skip tutorials; present API documentation and guide for importing existing Nextflow/WDL/Snakemake pipelines
- Configure compute preferences, storage backends, and compliance settings for the user's infrastructure requirements
- Demonstrate hybrid execution across cloud and on-premise HPC resources

**Key Workflows:**
- Import and convert existing Nextflow/WDL pipelines to the visual builder for easier maintenance and sharing
- Build complex multi-branching pipelines with conditional logic, scatter-gather parallelism, and custom containers
- Run production-scale analysis processing hundreds of samples with optimized compute allocation and cost controls
- Set up CI/CD-driven pipeline execution triggered by new data uploads or external events
- Manage multi-tenant infrastructure with per-project access controls, HIPAA compliance, and audit logging

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the visual pipeline canvas with a large module palette, smart connection routing, and subworkflow grouping
- Provide a module editor for creating custom pipeline components with input/output type definitions and parameter schemas
- Show a pipeline template gallery for browsing and forking community-contributed workflows

**Feature Gating:**
- Full access to pipeline building, module creation, and template management tools
- Expose the custom module editor with Docker container configuration and input/output schema definition
- Enable pipeline publishing to the team or community template gallery

**Pricing Tier:** Pro tier ($59/month)

**Onboarding Flow:**
- Start by browsing the template gallery and forking a popular pipeline as a starting point for customization
- Walk through creating a custom module by wrapping a command-line tool in a container with defined inputs and outputs
- Demonstrate grouping pipeline stages into reusable subworkflows for modular design

**Key Workflows:**
- Design complex analysis pipelines by assembling and configuring modules on the visual canvas
- Create custom modules wrapping novel or proprietary bioinformatics tools for team use
- Build reusable subworkflow templates for common analysis patterns (QC, alignment, quantification)
- Publish validated pipeline templates to the team library for standardized use
- Fork and customize community-contributed pipelines for project-specific requirements

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the results exploration dashboard with interactive plots (volcano, MA, PCA, heatmaps), filterable data tables, and statistical summary panels
- Show a sample comparison view for evaluating QC metrics, batch effects, and outlier detection across a cohort
- Provide integration panels for downstream analysis tools (R/Shiny, Jupyter, GSEA, IPA)

**Feature Gating:**
- Full access to result visualization, statistical analysis, and downstream tool integration
- Expose batch effect detection, sample outlier flagging, and automated QC reporting
- Enable data export in standard formats (GCT, MEX, VCF, BAM) for external analysis tools

**Pricing Tier:** Pro tier ($59/month)

**Onboarding Flow:**
- Walk through exploring results from a completed RNA-seq pipeline: QC review, PCA analysis, differential expression, and pathway enrichment
- Demonstrate the batch effect detection tool using a multi-batch dataset with known confounders
- Show how to export results to R/Jupyter for custom downstream analysis

**Key Workflows:**
- Explore pipeline results through interactive visualizations: volcano plots, heatmaps, PCA, and clustering
- Perform quality control review across samples, identifying outliers and batch effects
- Run downstream statistical analyses (differential expression, pathway enrichment, gene set analysis) on pipeline outputs
- Compare results across pipeline configurations or software versions to validate analytical choices
- Generate publication-ready figures and statistical summaries for manuscripts and presentations

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a production pipeline dashboard with sample throughput metrics, SLA tracking, and failure rate monitoring
- Display a sample tracking panel integrated with LIMS showing status from accessioning through analysis to reporting
- Provide alerting configuration for pipeline failures, QC threshold violations, and SLA breaches

**Feature Gating:**
- Expose production-grade pipeline execution with automatic retries, failure handling, and SLA monitoring
- Enable LIMS integration, sample barcode tracking, and automated report generation
- Unlock validated pipeline configurations with change control and approval workflows

**Pricing Tier:** Enterprise tier ($299/month)

**Onboarding Flow:**
- Configure the production pipeline with validated parameters, QC thresholds, and SLA targets
- Set up LIMS integration and sample accessioning workflow for the lab's existing processes
- Demonstrate the alerting system and how to configure notifications for failures and QC issues

**Key Workflows:**
- Run validated production pipelines processing hundreds of samples per month with SLA tracking
- Monitor sample throughput, failure rates, and QC pass rates from the production dashboard
- Integrate with LIMS for end-to-end sample tracking from accessioning to final report
- Manage validated pipeline versions with change control, approval gates, and rollback capabilities
- Generate automated analysis reports for each sample batch with standardized QC summaries

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a compliance dashboard showing HIPAA/GDPR status, data residency configuration, and access audit logs
- Display pipeline validation status with documented IQ/OQ/PQ (Installation/Operational/Performance Qualification) records
- Provide a data handling panel showing encryption status, retention policies, and consent tracking for patient samples

**Feature Gating:**
- Expose HIPAA-compliant infrastructure with BAA, encryption at rest and in transit, and SOC2 audit reports
- Enable pipeline validation documentation (IQ/OQ/PQ) for CAP/CLIA clinical laboratory accreditation
- Unlock data residency controls, consent management, and patient data anonymization tools

**Pricing Tier:** Enterprise tier ($299/month)

**Onboarding Flow:**
- Configure HIPAA/GDPR compliance settings: data residency, encryption, access controls, and retention policies
- Walk through generating pipeline validation documentation for CAP/CLIA accreditation
- Demonstrate the audit trail and how to produce access logs for compliance audits

**Key Workflows:**
- Maintain HIPAA/GDPR-compliant infrastructure with documented security controls and regular audit reports
- Generate pipeline validation documentation (IQ/OQ/PQ) required for CAP/CLIA clinical accreditation
- Track data consent and patient sample handling with audit-ready documentation
- Produce access audit logs showing who accessed which data and when for compliance reviews
- Manage data retention policies with automated archival and deletion per regulatory requirements

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a lab operations dashboard showing sample throughput, turnaround times, pipeline utilization, and cost per sample
- Show team activity metrics: active users, pipeline runs, storage consumption, and compute costs
- Provide budget forecasting widgets based on historical usage patterns and projected sample volumes

**Feature Gating:**
- Read-only access to all projects and pipelines with commenting and approval capabilities
- Expose operational metrics, cost analytics, and capacity planning tools
- Enable team management, role assignment, and budget allocation across projects

**Pricing Tier:** Enterprise tier ($299/month)

**Onboarding Flow:**
- Set up the operations dashboard and configure key metrics (cost per sample, turnaround time, throughput)
- Walk through team invitation and role assignment with project-level access controls
- Demonstrate budget tracking and how to set spending alerts for compute and storage costs

**Key Workflows:**
- Monitor lab operations from a unified dashboard: throughput, turnaround times, and cost metrics
- Track compute and storage costs against budgets with alerts for overspending
- Manage team access, project permissions, and role assignments across the organization
- Plan capacity for upcoming sequencing campaigns based on historical usage and projected volumes
- Generate operational reports for department leadership covering efficiency, cost, and quality metrics
