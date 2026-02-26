# NeuralForge — Persona Subspecs

> Parent spec: [`specs/23-neuralforge.md`](../23-neuralforge.md)

---

## Experience-Level Personas

### Student/Academic

**Modified UI/UX:**
- Display a simplified node palette with common layer types (Dense, Conv2D, LSTM, Dropout) and hide advanced/experimental layers behind a "Show All" toggle
- Show real-time tensor shape annotations on every connection with color-coded warnings when shapes are incompatible
- Provide a "Concept Mode" that labels each node with its mathematical operation (e.g., "matrix multiply + bias + ReLU") alongside the layer name

**Feature Gating:**
- Allow up to 5 concurrent projects and 10 GPU-hours/month of training on T4 instances
- Expose the visual builder, basic experiment tracking, and ONNX export; hide hyperparameter sweep, distributed training, and model serving
- Provide access to public model zoo architectures (ResNet, BERT, GPT-2 small) for learning and modification

**Pricing Tier:** Free tier

**Onboarding Flow:**
- First-run tutorial walks through building a simple MNIST classifier by dragging Conv2D, MaxPool, and Dense layers onto the canvas
- Show an interactive shape propagation animation to teach how tensor dimensions transform through layers
- Offer a curated "ML 101" project gallery with pre-built architectures and datasets for each major deep learning concept

**Key Workflows:**
- Build a neural network from scratch using drag-and-drop layers and verify shapes before training
- Train a model on a provided dataset (MNIST, CIFAR-10, IMDB) and observe live loss/accuracy curves
- Load a pre-built architecture from the model zoo, modify it, and compare training results
- Export a trained model to ONNX format for a class project demonstration

---

### Junior Engineer (0-3 years)

**Modified UI/UX:**
- Show the full node palette with categorized layers and an integrated search; display parameter recommendations based on common best practices
- Provide a "Training Coach" sidebar that monitors training in real-time and flags issues (vanishing gradients, overfitting, learning rate too high)
- Display experiment comparison views with side-by-side metric plots and architecture diffs

**Feature Gating:**
- Unlock 50 GPU-hours/month on T4/A10G instances, hyperparameter sweep (grid and random search), and multi-format export (ONNX, TFLite, CoreML)
- Expose collaboration features for sharing projects and getting team feedback on architectures
- Hide distributed training, custom operator support, and model serving infrastructure

**Pricing Tier:** Pro tier ($49/month)

**Onboarding Flow:**
- Guide the user through importing a dataset, building an architecture from a template, training, and evaluating with a focus on the experiment tracking workflow
- Demonstrate the hyperparameter sweep feature by running 5 variants of a model simultaneously
- Show how to use the "Training Coach" to diagnose and fix a deliberately misconfigured training run

**Key Workflows:**
- Design and iterate on model architectures for production tasks (object detection, text classification, time series)
- Run hyperparameter sweeps and compare results across experiments to find the best configuration
- Use the Training Coach to diagnose training issues and apply recommended fixes
- Export trained models in deployment-ready formats (TFLite for mobile, CoreML for iOS, ONNX for cross-platform)
- Share architecture designs with senior engineers for review and feedback

---

### Senior/Expert

**Modified UI/UX:**
- Provide a code-alongside-canvas view where the visual graph auto-generates PyTorch/TensorFlow code that can be edited directly
- Expose the full experiment management system with custom metrics, artifact versioning, and reproducibility checksums
- Support custom node creation via Python for defining novel layers and loss functions within the visual environment

**Feature Gating:**
- Unlimited GPU access (A10G/A100), distributed training across multiple GPUs, and custom training loop overrides
- Expose model serving with A/B testing, canary deployment, and auto-scaling inference endpoints
- Enable team management, SSO, priority GPU queuing, and on-premise deployment options

**Pricing Tier:** Enterprise tier ($199/month or annual contract)

**Onboarding Flow:**
- Skip basic tutorials; present an import wizard for existing PyTorch/TensorFlow projects that converts code to visual graphs
- Configure GPU preferences, default training infrastructure, and experiment tracking integrations (W&B, MLflow)
- Demonstrate the custom node API and code-canvas synchronization workflow

**Key Workflows:**
- Design novel architectures combining standard and custom layers, with real-time code generation
- Run distributed training across multiple A100 GPUs with advanced scheduling and checkpointing
- Manage hundreds of experiments with custom metrics, artifact versioning, and automated comparison
- Deploy models to production with A/B testing, monitoring, and auto-scaling inference endpoints
- Build reusable architecture templates and custom training pipelines for the team

---

## Role-Based Personas

### Designer/Creator

**Modified UI/UX:**
- Emphasize the visual canvas with aesthetic layout options, node grouping, and color-coded subgraph annotations
- Provide architecture templates for common tasks (image classification, object detection, NLP, generative models) as starting points
- Show a live architecture summary panel (total parameters, FLOPs, memory estimate, latency estimate) updating as the graph is modified

**Feature Gating:**
- Full access to the visual builder, all layer types, and architecture templates
- Expose model export in all formats; hide training infrastructure and experiment management
- Enable architecture sharing and publishing to the community model zoo

**Pricing Tier:** Pro tier ($49/month)

**Onboarding Flow:**
- Start by selecting a task type (vision, NLP, tabular, generative) and browsing relevant architecture templates
- Demonstrate how to customize a template by swapping layers, adjusting parameters, and observing shape changes
- Show how to publish a designed architecture to the team or community model zoo

**Key Workflows:**
- Design neural network architectures from scratch or by modifying templates for specific tasks
- Validate architecture feasibility by checking parameter counts, memory requirements, and estimated inference latency
- Create architecture variants and compare their theoretical properties before training
- Publish polished architecture diagrams for documentation, papers, or presentations
- Build a library of reusable architecture components (attention blocks, encoder-decoder patterns)

---

### Analyst/Simulator

**Modified UI/UX:**
- Foreground the experiment tracking dashboard with sortable/filterable tables, metric comparison charts, and statistical analysis tools
- Show training diagnostics prominently: gradient distributions, activation histograms, loss landscape visualizations
- Provide an "Experiment Notebook" panel for documenting hypotheses, observations, and conclusions alongside each run

**Feature Gating:**
- Full access to experiment tracking, metric logging, hyperparameter sweep, and result analysis tools
- Expose advanced diagnostics (gradient flow analysis, layer-wise learning rate visualization, overfitting detection)
- Enable data export to Jupyter notebooks and integration with external analysis tools

**Pricing Tier:** Pro tier ($49/month)

**Onboarding Flow:**
- Walk through a full experiment cycle: define hypothesis, configure sweep, launch runs, analyze results, and document findings
- Demonstrate the metric comparison tool by loading results from multiple training runs
- Show how to use gradient diagnostics to identify and fix training instabilities

**Key Workflows:**
- Design and execute systematic hyperparameter searches across architectures
- Analyze training dynamics using gradient flow visualization and activation statistics
- Compare experiments across multiple dimensions (architecture, hyperparameters, data augmentation) in a unified view
- Generate reproducible experiment reports with linked artifacts, metrics, and environment specifications
- Export analysis results and trained model metadata to external tools for further investigation

---

### Manufacturing/Process

**Modified UI/UX:**
- Show a model deployment pipeline view: training, validation, optimization (quantization/pruning), packaging, and serving
- Display target device compatibility matrices and performance benchmarks for each export format
- Provide a model optimization panel with quantization, pruning, and distillation tools with before/after accuracy comparisons

**Feature Gating:**
- Full access to model export, optimization (quantization, pruning, distillation), and deployment tools
- Expose CI/CD pipeline integration for automated model validation and deployment
- Enable model registry with version control, approval gates, and rollback capabilities

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Walk through the end-to-end deployment pipeline: take a trained model, quantize it, benchmark on target hardware, and deploy to an endpoint
- Demonstrate the model registry and how to manage model versions across development, staging, and production
- Show CI/CD integration for automated testing of model quality before deployment

**Key Workflows:**
- Optimize trained models for production deployment via quantization, pruning, and knowledge distillation
- Benchmark model performance (latency, throughput, accuracy) across target hardware platforms
- Manage a model registry with versioning, approval workflows, and production rollback capabilities
- Set up CI/CD pipelines that automatically validate model quality before promoting to production
- Monitor deployed models for drift, latency degradation, and accuracy changes

---

### Regulatory/Compliance

**Modified UI/UX:**
- Add a model card editor with fields for intended use, limitations, fairness metrics, and known failure modes
- Show a data lineage view tracing from training data through preprocessing to model output
- Display fairness and bias analysis dashboards with demographic parity, equalized odds, and calibration metrics

**Feature Gating:**
- Expose model interpretability tools: SHAP values, attention maps, feature importance rankings
- Enable comprehensive audit logging of all training runs, data access, and model modifications
- Unlock fairness testing, bias detection, and automated model card generation

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Set up a compliance-ready workspace with mandatory model card completion before deployment approval
- Walk through generating a model card with fairness metrics for a sample classification model
- Demonstrate the audit trail and how to trace any production prediction back to its training data and parameters

**Key Workflows:**
- Generate comprehensive model cards documenting training data, architecture decisions, and performance characteristics
- Run fairness and bias audits across protected demographic groups and document results
- Maintain a complete audit trail linking every deployed model to its exact training data, code, and hyperparameters
- Produce explainability reports using SHAP/attention visualization for regulatory review
- Ensure data lineage documentation satisfies GDPR and emerging AI regulation requirements

---

### Manager/Decision-maker

**Modified UI/UX:**
- Present a portfolio view of all ML projects with status (research, training, deployed), team assignments, and key metrics
- Show resource utilization dashboards: GPU hours consumed, cost per experiment, team activity
- Provide ROI-focused views comparing model accuracy improvements against compute costs

**Feature Gating:**
- Read-only access to all projects with commenting and approval capabilities
- Expose cost analytics, team management, and resource allocation tools
- Enable project prioritization and GPU budget allocation across teams

**Pricing Tier:** Enterprise tier ($199/month)

**Onboarding Flow:**
- Set up the portfolio dashboard and configure project categories and team structures
- Walk through the cost tracking and GPU budget allocation workflow
- Demonstrate the approval workflow for promoting models from development to production

**Key Workflows:**
- Monitor all ML projects across teams from a unified dashboard with key performance indicators
- Allocate GPU budgets across projects and track utilization against spending limits
- Review and approve model deployments based on accuracy, fairness, and cost metrics
- Generate executive reports on ML team productivity, model performance trends, and infrastructure costs
- Plan capacity and budget for upcoming quarters based on historical usage and projected needs
