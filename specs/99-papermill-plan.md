# 99. PaperMill — Pulp and Paper Process Simulation Platform

## Implementation Plan

**MVP Scope:** Browser-based flowsheet editor with drag-and-drop placement of 25+ pulp and paper unit operations (digester, washer, screen, bleach tower, headbox, wire section, press section, dryer, mixer, splitter, heater, stock chest, fan pump) rendered via SVG with smart connection routing, custom Rust solver implementing rigorous mass/energy balance tracking fiber (bone-dry tons/day), water (m³/day), dissolved solids (kg/day), and 10 key chemical species (NaOH, Na2S, ClO2, O2, H2O2, Na2SO4, CaCO3, CaCl2, lignin, hexenuronic acid) across up to 100 interconnected streams, kraft cooking model with Purdue/H-factor kinetics and kappa number prediction (target ±2 kappa units vs. mill data), brownstock washing with Norden efficiency factors and dilution factor optimization, ECF bleach plant simulation (D0-EOP-D1 sequence) with brightness and viscosity prediction, paper machine drainage/pressing/drying physics (Kozeny-Carman drainage, Carlsson pressing consolidation, multicylinder drying heat transfer) predicting moisture profile from headbox (99.0% water) to reel (6-8% moisture), stream table viewer with tabular and Sankey diagram visualization of flows, interactive property plots (kappa vs. H-factor, moisture vs. dryer position, brightness vs. ClO2 charge), PDF/Excel report generation, pre-built kraft mill template (wood yard → digester → brownstock washing → O2 delignification → bleach plant → paper machine → finished reel), PostgreSQL for flowsheet storage + S3 for simulation results, Stripe billing with Free / Pro $129/mo / Mill Engineer $299/user/mo.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend API | Rust (Axum 0.7) | Tokio async runtime, Tower middleware, JWT auth |
| Mass/Energy Solver | Rust (native server) | Sequential modular with tear streams, Newton-Raphson for recycle convergence |
| Cooking Kinetics | Rust (`ode_solvers` crate) | Purdue H-factor integration, Vroom delignification rate ODE |
| Paper Machine Physics | Rust (custom PDEs) | Kozeny-Carman drainage, Carlsson pressing, Campbell/Watters drying |
| Optimization Engine | Python 3.12 (FastAPI) | SciPy minimize for chemical charge optimization, yield maximization |
| Database | PostgreSQL 16 | Flowsheets, users, simulations, fiber property library |
| ORM / Query | SQLx (Rust) | Compile-time checked queries, raw SQL migrations |
| Auth | Custom JWT + OAuth 2.0 | Google and GitHub OAuth providers, bcrypt password hashing |
| Object Storage | AWS S3 | Simulation results (stream tables, time series), PDF reports, flowsheet snapshots |
| Frontend Framework | React 19 + TypeScript | Vite build, Zustand state management |
| Flowsheet Editor | Custom SVG renderer | Dagre.js for auto-layout, snap-to-grid, stream routing |
| Visualization | D3.js v7 | Sankey diagrams (mass/energy flows), trend plots, property curves |
| Job Queue | Redis 7 + Tokio tasks | Server-side simulation job management |
| Billing | Stripe | Checkout Sessions, Customer Portal, Webhooks |
| CDN | CloudFront | Static frontend assets, PDF report delivery |
| Monitoring | Prometheus + Grafana, Sentry | Infrastructure metrics, error tracking |
| CI/CD | GitHub Actions | Rust tests, Docker image push, migration validation |

### Key Architecture Decisions

1. **Sequential modular solver with tear stream iteration rather than equation-oriented**: Pulp and paper flowsheets have strong recycle streams (filtrate recycle, white water loops, chemical recovery cycle). Sequential modular solves each unit operation independently with tear streams (initial guesses for recycle inputs) and iterates until convergence. This matches how engineers think (follow the flow) and allows swapping unit operation models without rebuilding a global Jacobian. Convergence is achieved via Wegstein acceleration or Newton-Raphson on tear variables. Equation-oriented (solving all equations simultaneously) was rejected because it requires auto-differentiation for 1000+ variables and makes debugging harder.

2. **Rust solver for mass/energy balance rather than Python/DWSIM**: Rust provides memory safety, zero-cost abstractions, and excellent parallelization for multi-stream flowsheets. Python tools like DWSIM/Aspen Plus clone are too slow for real-time interactive simulation and require massive libraries. Our Rust solver compiles unit operation models directly with full control over thermodynamics (black liquor boiling point rise, steam tables, fiber-water interaction). The solver is 50-100x faster than equivalent Python for large kraft mills (500+ streams).

3. **Kraft cooking H-factor model rather than simplified yield correlations**: The Purdue kinetics model integrates H-factor (time-temperature severity) and predicts kappa number (lignin content), carbohydrate degradation, and pulp yield via ODEs. This is the industry standard and matches mill data within ±2 kappa units. Simplified correlations (linear kappa vs. cooking time) were rejected because they fail for varying wood species, liquor-to-wood ratio, and EA (effective alkali) charge. We implement Vroom's delignification rate ODE with species-specific parameters for softwood/hardwood.

4. **Paper machine section-by-section physics rather than empirical overall model**: The paper machine is split into headbox (flow distribution), wire section (drainage via Kozeny-Carman for gravity/vacuum dewatering), press section (Carlsson/Wahlström consolidation model for nip pressing), and dryer section (heat/mass transfer through porous fiber mats). This gives accurate moisture profile prediction (99% → 6% step by step) and allows troubleshooting individual sections. Empirical "overall moisture removal" models were rejected because they can't diagnose problems like poor drainage on the wire or insufficient pressing impulse.

5. **Sankey diagram for mass flows rather than table-only**: Sankey diagrams visualize fiber, water, and chemical flows as proportional-width ribbons, making mass balance errors and major flows instantly visible. Engineers spot issues like "white water overflow = 3x design" or "ClO2 consumption > expected" at a glance. This complements tabular stream tables which are still needed for precise numbers. D3.js renders interactive Sankeys with hover tooltips and drill-down to individual species (e.g., show just NaOH flow or just fiber flow).

---

## Database Schema

### SQL Migrations

```sql
-- migrations/001_initial.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    name TEXT NOT NULL,
    avatar_url TEXT,
    auth_provider TEXT DEFAULT 'email',
    auth_provider_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX users_stripe_idx ON users(stripe_customer_id);

-- Organizations
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan TEXT NOT NULL DEFAULT 'free',
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX orgs_slug_idx ON organizations(slug);

-- Organization Members
CREATE TABLE org_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, user_id)
);
CREATE INDEX org_members_org_idx ON org_members(org_id);

-- Flowsheets
CREATE TABLE flowsheets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    flowsheet_type TEXT NOT NULL DEFAULT 'kraft_mill',
    flowsheet_data JSONB NOT NULL DEFAULT '{}',
    settings JSONB DEFAULT '{}',
    is_public BOOLEAN NOT NULL DEFAULT false,
    forked_from UUID REFERENCES flowsheets(id) ON DELETE SET NULL,
    thumbnail_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX flowsheets_owner_idx ON flowsheets(owner_id);
CREATE INDEX flowsheets_type_idx ON flowsheets(flowsheet_type);
CREATE INDEX flowsheets_updated_idx ON flowsheets(updated_at DESC);

-- Simulations
CREATE TABLE simulations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    flowsheet_id UUID NOT NULL REFERENCES flowsheets(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending',
    convergence_method TEXT NOT NULL DEFAULT 'wegstein',
    tear_streams JSONB DEFAULT '[]',
    max_iterations INTEGER NOT NULL DEFAULT 100,
    tolerance REAL NOT NULL DEFAULT 1e-4,
    results_url TEXT,
    results_summary JSONB,
    convergence_history JSONB,
    error_message TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    wall_time_ms INTEGER,
    iterations_count INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX sims_flowsheet_idx ON simulations(flowsheet_id);
CREATE INDEX sims_status_idx ON simulations(status);

-- Simulation Jobs
CREATE TABLE simulation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    worker_id TEXT,
    priority INTEGER DEFAULT 0,
    progress_pct REAL DEFAULT 0.0,
    current_iteration INTEGER DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX jobs_sim_idx ON simulation_jobs(simulation_id);

-- Fiber Properties
CREATE TABLE fiber_properties (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    species TEXT,
    density REAL,
    moisture_content REAL,
    lignin_content REAL,
    hemicellulose_content REAL,
    cellulose_content REAL,
    extractives_content REAL,
    fiber_length_avg REAL,
    fiber_coarseness REAL,
    cooking_params JSONB,
    is_builtin BOOLEAN DEFAULT true,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX fiber_category_idx ON fiber_properties(category);

-- Chemicals
CREATE TABLE chemicals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    formula TEXT NOT NULL,
    cas_number TEXT,
    molecular_weight REAL NOT NULL,
    category TEXT,
    unit_price REAL,
    currency TEXT DEFAULT 'USD',
    supplier TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX chemicals_formula_idx ON chemicals(formula);

-- Reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    simulation_id UUID NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
    report_type TEXT NOT NULL,
    file_url TEXT NOT NULL,
    file_size_bytes INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reports_sim_idx ON reports(simulation_id);

-- Usage Tracking
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    record_type TEXT NOT NULL,
    quantity REAL NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX usage_user_period_idx ON usage_records(user_id, period_start, period_end);
```

### Rust SQLx Structs

```rust
// src/db/models.rs

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, FromRow, Serialize)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    #[sqlx(skip)]
    #[serde(skip)]
    pub password_hash: Option<String>,
    pub name: String,
    pub avatar_url: Option<String>,
    pub auth_provider: String,
    pub auth_provider_id: Option<String>,
    pub plan: String,
    pub stripe_customer_id: Option<String>,
    pub stripe_subscription_id: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Flowsheet {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub org_id: Option<Uuid>,
    pub name: String,
    pub description: String,
    pub flowsheet_type: String,
    pub flowsheet_data: serde_json::Value,
    pub settings: serde_json::Value,
    pub is_public: bool,
    pub forked_from: Option<Uuid>,
    pub thumbnail_url: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, FromRow, Serialize)]
pub struct Simulation {
    pub id: Uuid,
    pub flowsheet_id: Uuid,
    pub user_id: Uuid,
    pub status: String,
    pub convergence_method: String,
    pub tear_streams: serde_json::Value,
    pub max_iterations: i32,
    pub tolerance: f32,
    pub results_url: Option<String>,
    pub results_summary: Option<serde_json::Value>,
    pub convergence_history: Option<serde_json::Value>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub wall_time_ms: Option<i32>,
    pub iterations_count: Option<i32>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct MaterialStream {
    pub fiber_flow: f64,
    pub water_flow: f64,
    pub dissolved_solids: f64,
    pub species: Vec<SpeciesConcentration>,
    pub temperature: f64,
    pub pressure: f64,
    pub consistency: f64,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct SpeciesConcentration {
    pub formula: String,
    pub mass_flow: f64,
}
```

---

## Solver Architecture Deep-Dive

### Governing Equations and Sequential Modular Approach

PaperMill's solver uses **Sequential Modular (SM)** simulation, where each unit operation is solved independently in a specific sequence, and tear streams are iterated until global mass/energy balance convergence.

**Mass balance for each unit operation:**
```
For fiber:    Σ(inlet_fiber) = Σ(outlet_fiber)
For water:    Σ(inlet_water) = Σ(outlet_water)
For species i: Σ(inlet_species_i) + generation_i - consumption_i = Σ(outlet_species_i)
```

**Energy balance:**
```
Σ(inlet_enthalpy) + Q_in + W_in = Σ(outlet_enthalpy) + Q_out + W_out

where enthalpy H = m·cp·(T - Tref) for water
                  + m·H_evap for steam
                  + m·cp_fiber·(T - Tref) for fiber
```

**Kraft Cooking (Purdue/H-factor model):**

```
dK/dt = -k₁(T,EA) · K    (bulk delignification)
dK/dt = -k₂(T,EA) · K    (residual delignification, K < K_transition)

H-factor: H = ∫ exp((43.2 - 16113/T_K)) dt
```

For softwood kraft:
- Initial kappa: K₀ ≈ 100-120
- Target kappa: 25-35
- Typical H-factor: 1000-1500
- EA charge: 16-20%

**Paper Machine Drainage (Kozeny-Carman):**

```
Drainage rate: dW/dt = (ΔP · A) / (μ · R_specific · m_fiber)

where R_specific = k · (1 - ε)² / ε³ · S²
      ε = void fraction (porosity)
      S = specific surface area (m²/kg)
```

**Press Section (Carlsson Consolidation):**

```
Final moisture M_f = M_0 · exp(-k_press · P_impulse)

where P_impulse = ∫ P(t) dt
```

**Dryer Section (Heat/Mass Transfer):**

```
Heat transfer: Q = U·A·LMTD
Mass transfer: dM/dt = -Q / H_evap
```

### Convergence Acceleration

**Wegstein Acceleration:**
```
x_(k+1) = x_k + q · (x_new - x_k)
where q = s / (s - 1)
      s = (x_new - x_new_prev) / (x_k - x_k_prev)
```

**Newton-Raphson:**
```
J · Δx = -R
x_(k+1) = x_k + Δx
```

---

## Architecture Deep-Dives

### 1. Simulation API Handler

```rust
// src/api/handlers/simulation.rs

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use uuid::Uuid;
use crate::{
    db::models::{Simulation, Flowsheet},
    solver::flowsheet::validate_flowsheet,
    state::AppState,
    auth::Claims,
    error::ApiError,
};

pub async fn create_simulation(
    State(state): State<AppState>,
    claims: Claims,
    Path(flowsheet_id): Path<Uuid>,
    Json(req): Json<CreateSimulationRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let flowsheet = sqlx::query_as!(
        Flowsheet,
        "SELECT * FROM flowsheets WHERE id = $1 AND (owner_id = $2 OR org_id IN (
            SELECT org_id FROM org_members WHERE user_id = $2
        ))",
        flowsheet_id,
        claims.user_id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound("Flowsheet not found"))?;

    let validation = validate_flowsheet(&flowsheet.flowsheet_data)?;
    if !validation.is_valid {
        return Err(ApiError::ValidationError(validation.errors.join(", ")));
    }

    let user = sqlx::query_as!(
        crate::db::models::User,
        "SELECT * FROM users WHERE id = $1",
        claims.user_id
    )
    .fetch_one(&state.db)
    .await?;

    let unit_count = validation.unit_count;
    if user.plan == "free" && unit_count > 10 {
        return Err(ApiError::PlanLimit(
            "Free plan supports up to 10 unit operations. Upgrade to Pro."
        ));
    }

    let tear_streams = validation.tear_streams;
    let max_iterations = req.max_iterations.unwrap_or(100);
    let tolerance = req.tolerance.unwrap_or(1e-4);

    let sim = sqlx::query_as!(
        Simulation,
        r#"INSERT INTO simulations
            (flowsheet_id, user_id, status, convergence_method, tear_streams,
             max_iterations, tolerance)
        VALUES ($1, $2, 'pending', $3, $4, $5, $6)
        RETURNING *"#,
        flowsheet_id,
        claims.user_id,
        req.convergence_method,
        serde_json::to_value(&tear_streams)?,
        max_iterations,
        tolerance,
    )
    .fetch_one(&state.db)
    .await?;

    let priority = if user.plan == "mill_engineer" { 10 } else { 0 };
    let job = sqlx::query_as!(
        crate::db::models::SimulationJob,
        r#"INSERT INTO simulation_jobs (simulation_id, priority)
        VALUES ($1, $2) RETURNING *"#,
        sim.id,
        priority,
    )
    .fetch_one(&state.db)
    .await?;

    state.redis
        .publish("simulation:jobs", serde_json::to_string(&job.id)?)
        .await?;

    Ok((StatusCode::CREATED, Json(sim)))
}
```

### 2. Kraft Digester Unit Operation

```rust
// solver/src/units/digester.rs

use crate::streams::MaterialStream;
use std::error::Error;

pub struct KraftDigester {
    pub name: String,
    pub volume: f64,
    pub temperature_profile: Vec<(f64, f64)>,
    pub liquor_to_wood: f64,
    pub ea_charge: f64,
    pub sulfidity: f64,
}

#[derive(Debug)]
pub struct CookingResult {
    pub kappa_number: f64,
    pub yield: f64,
    pub h_factor: f64,
    pub residual_ea: f64,
    pub cooking_time: f64,
}

impl KraftDigester {
    pub fn solve(
        &self,
        wood_inlet: &MaterialStream,
        liquor_inlet: &MaterialStream,
    ) -> Result<(MaterialStream, CookingResult), Box<dyn Error>> {
        let wood_flow = wood_inlet.fiber_flow;
        let initial_kappa = 110.0;

        let h_factor = self.calculate_h_factor();
        let (final_kappa, cooking_time) = self.purdue_kinetics(
            initial_kappa,
            h_factor,
            self.ea_charge,
        )?;

        let yield_pct = self.calculate_yield(final_kappa, h_factor);
        let pulp_flow = wood_flow * yield_pct / 100.0;

        let naoh_consumed = wood_flow * self.ea_charge / 100.0 * 1000.0;
        let na2s_consumed = naoh_consumed * self.sulfidity / 100.0 * (78.04 / 40.0);

        let initial_naoh = liquor_inlet.get_species_flow("NaOH");
        let residual_ea = (initial_naoh - naoh_consumed) / (wood_flow * self.liquor_to_wood);

        let mut pulp_outlet = MaterialStream::new();
        pulp_outlet.fiber_flow = pulp_flow;
        pulp_outlet.water_flow = wood_flow * self.liquor_to_wood * 1.0;
        pulp_outlet.consistency = (pulp_flow / (pulp_flow + pulp_outlet.water_flow)) * 100.0;
        pulp_outlet.temperature = 95.0;
        pulp_outlet.add_species("NaOH", initial_naoh - naoh_consumed);
        pulp_outlet.add_species("Na2S", liquor_inlet.get_species_flow("Na2S") - na2s_consumed);
        pulp_outlet.add_species("Lignin", pulp_flow * final_kappa / 100.0 * 10.0);

        let result = CookingResult {
            kappa_number: final_kappa,
            yield: yield_pct,
            h_factor,
            residual_ea,
            cooking_time,
        };

        Ok((pulp_outlet, result))
    }

    fn calculate_h_factor(&self) -> f64 {
        let mut h = 0.0;
        for i in 0..self.temperature_profile.len() - 1 {
            let (t1, temp1) = self.temperature_profile[i];
            let (t2, temp2) = self.temperature_profile[i + 1];
            let dt = t2 - t1;
            let temp_avg = (temp1 + temp2) / 2.0;
            let temp_k = temp_avg + 273.15;
            let integrand = f64::exp(43.2 - 16113.0 / temp_k);
            h += integrand * dt;
        }
        h
    }

    fn purdue_kinetics(&self, k0: f64, h_target: f64, ea: f64)
        -> Result<(f64, f64), Box<dyn Error>> {
        let r_base = 0.0025;
        let r = r_base * (1.0 + 0.05 * (ea - 18.0));

        let k_bulk_final = 35.0;
        let h_bulk = if k0 > k_bulk_final {
            -(k0 / k_bulk_final).ln() / r
        } else {
            0.0
        };

        let k_final = if h_target > h_bulk {
            let h_residual = h_target - h_bulk;
            k_bulk_final * f64::exp(-r / 3.0 * h_residual)
        } else {
            k0 * f64::exp(-r * h_target)
        };

        let temp_k = 170.0 + 273.15;
        let integrand_170 = f64::exp(43.2 - 16113.0 / temp_k);
        let cooking_time = h_target / integrand_170;

        Ok((k_final, cooking_time))
    }

    fn calculate_yield(&self, kappa: f64, h_factor: f64) -> f64 {
        let yield_base = 50.0;
        let yield_kappa_penalty = -0.15 * (30.0 - kappa);
        let yield_h_penalty = -0.002 * (h_factor - 1200.0);
        (yield_base + yield_kappa_penalty + yield_h_penalty).max(40.0).min(55.0)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_kraft_digester_softwood() {
        let digester = KraftDigester {
            name: "Continuous Digester".to_string(),
            volume: 1500.0,
            temperature_profile: vec![
                (0.0, 100.0), (1.0, 140.0), (2.0, 170.0),
                (4.0, 170.0), (5.0, 95.0),
            ],
            liquor_to_wood: 4.0,
            ea_charge: 18.0,
            sulfidity: 30.0,
        };

        let mut wood_inlet = MaterialStream::new();
        wood_inlet.fiber_flow = 1000.0;

        let mut liquor_inlet = MaterialStream::new();
        liquor_inlet.add_species("NaOH", 800.0);
        liquor_inlet.add_species("Na2S", 250.0);

        let (pulp, result) = digester.solve(&wood_inlet, &liquor_inlet).unwrap();

        assert!(result.kappa_number > 26.0 && result.kappa_number < 34.0);
        assert!(result.yield > 46.0 && result.yield < 54.0);
        assert!(result.h_factor > 1000.0 && result.h_factor < 1400.0);
    }
}
```

### 3. Paper Machine Wire Section

```rust
// solver/src/units/wire_section.rs

use crate::streams::MaterialStream;
use std::error::Error;

pub struct WireSection {
    pub name: String,
    pub wire_length: f64,
    pub wire_width: f64,
    pub wire_speed: f64,
    pub elements: Vec<DrainageElement>,
}

#[derive(Debug, Clone)]
pub enum DrainageElement {
    TableRoll { vacuum_kpa: f64 },
    Foil { angle_deg: f64 },
    VacuumBox { vacuum_kpa: f64, slot_width_mm: f64 },
    Suction { vacuum_kpa: f64 },
}

impl WireSection {
    pub fn solve(&self, headbox_inlet: &MaterialStream)
        -> Result<(MaterialStream, DrainageProfile), Box<dyn Error>> {
        let initial_consistency = headbox_inlet.consistency;
        if initial_consistency < 0.3 || initial_consistency > 2.0 {
            return Err("Headbox consistency out of range (0.3-2.0%)".into());
        }

        let mut current_consistency = initial_consistency;
        let mut position = 0.0;
        let mut profile = DrainageProfile { points: vec![] };

        let fiber_flow_bdt_per_min = headbox_inlet.fiber_flow / (24.0 * 60.0);

        profile.points.push((position, current_consistency));

        for element in &self.elements {
            let (delta_consistency, delta_position) = self.calculate_drainage(
                element,
                current_consistency,
                fiber_flow_bdt_per_min,
            );

            current_consistency += delta_consistency;
            position += delta_position;
            profile.points.push((position, current_consistency));

            if current_consistency > 25.0 {
                current_consistency = 25.0;
                break;
            }
        }

        let fiber_flow = headbox_inlet.fiber_flow;
        let water_flow_after_wire = fiber_flow * (100.0 / current_consistency - 1.0);

        let mut pulp_outlet = MaterialStream::new();
        pulp_outlet.fiber_flow = fiber_flow;
        pulp_outlet.water_flow = water_flow_after_wire;
        pulp_outlet.consistency = current_consistency;
        pulp_outlet.temperature = headbox_inlet.temperature - 2.0;

        Ok((pulp_outlet, profile))
    }

    fn calculate_drainage(&self, element: &DrainageElement,
        current_consistency: f64, _fiber_flow: f64) -> (f64, f64) {
        let specific_resistance = self.kozeny_carman_resistance(current_consistency);
        let delta_position = 0.5;

        let drainage_rate = match element {
            DrainageElement::TableRoll { vacuum_kpa } => {
                let delta_p = vacuum_kpa * 1000.0;
                let drainage = delta_p / (specific_resistance * 1e9);
                drainage.min(0.5)
            },
            DrainageElement::Foil { angle_deg } => {
                let gravity_assist = angle_deg / 45.0;
                let drainage = 0.3 * gravity_assist;
                drainage.min(0.4)
            },
            DrainageElement::VacuumBox { vacuum_kpa, slot_width_mm } => {
                let delta_p = vacuum_kpa * 1000.0;
                let slot_factor = slot_width_mm / 10.0;
                let drainage = delta_p / (specific_resistance * 1e9) * slot_factor;
                drainage.min(1.5)
            },
            DrainageElement::Suction { vacuum_kpa } => {
                let delta_p = vacuum_kpa * 1000.0;
                let drainage = delta_p / (specific_resistance * 1e9);
                drainage.min(1.2)
            },
        };

        (drainage_rate, delta_position)
    }

    fn kozeny_carman_resistance(&self, consistency: f64) -> f64 {
        let porosity = 1.0 - (consistency / 100.0) / 1.5;
        let specific_surface = 200.0;
        let kozeny_const = 5.0;
        kozeny_const * (1.0 - porosity).powi(2) / porosity.powi(3) * specific_surface.powi(2)
    }
}

#[derive(Debug)]
pub struct DrainageProfile {
    pub points: Vec<(f64, f64)>,
}
```

### 4. Sequential Modular Solver

```rust
// solver/src/sequential_modular.rs

use std::collections::HashMap;

pub struct SequentialModularSolver {
    pub units: Vec<Box<dyn UnitOperation>>,
    pub connections: Vec<StreamConnection>,
    pub tear_streams: Vec<String>,
    pub convergence_method: ConvergenceMethod,
    pub max_iterations: usize,
    pub tolerance: f64,
}

impl SequentialModularSolver {
    pub fn solve(&mut self) -> Result<SolverResult, Box<dyn Error>> {
        let mut streams: HashMap<String, MaterialStream> = HashMap::new();
        let mut tear_guesses: HashMap<String, MaterialStream> = HashMap::new();
        let mut convergence_history = vec![];

        for tear_id in &self.tear_streams {
            tear_guesses.insert(tear_id.clone(), MaterialStream::default_guess());
        }

        let mut prev_tear_values: Option<HashMap<String, f64>> = None;
        let mut prev_new_values: Option<HashMap<String, f64>> = None;

        for iteration in 0..self.max_iterations {
            for (tear_id, guess) in &tear_guesses {
                streams.insert(tear_id.clone(), guess.clone());
            }

            let mut unit_results = HashMap::new();
            for unit in &self.units {
                let inlets = self.get_unit_inlets(unit.id(), &streams)?;
                let result = unit.solve(&inlets)?;

                for (outlet_name, outlet_stream) in &result.outlets {
                    let stream_id = format!("{}:{}", unit.id(), outlet_name);
                    streams.insert(stream_id, outlet_stream.clone());
                }

                unit_results.insert(unit.id().to_string(), result);
            }

            let mut new_tear_values = HashMap::new();
            let mut new_tear_streams = HashMap::new();
            for tear_id in &self.tear_streams {
                if let Some(stream) = streams.get(tear_id) {
                    new_tear_streams.insert(tear_id.clone(), stream.clone());
                    new_tear_values.insert(tear_id.clone(), stream.fiber_flow);
                }
            }

            let residual = self.calculate_residual(&tear_guesses, &new_tear_streams);

            let tear_value_snapshot: HashMap<String, f64> = tear_guesses.iter()
                .map(|(k, v)| (k.clone(), v.fiber_flow))
                .collect();
            convergence_history.push(ConvergencePoint {
                iteration,
                residual,
                tear_values: tear_value_snapshot,
            });

            if residual < self.tolerance {
                return Ok(SolverResult {
                    converged: true,
                    iterations: iteration + 1,
                    streams,
                    unit_results,
                    convergence_history,
                });
            }

            match self.convergence_method {
                ConvergenceMethod::DirectSubstitution => {
                    tear_guesses = new_tear_streams;
                },
                ConvergenceMethod::Wegstein => {
                    if let (Some(prev_tear), Some(prev_new)) =
                        (&prev_tear_values, &prev_new_values) {
                        tear_guesses = self.wegstein_update(
                            &tear_guesses, &new_tear_streams,
                            prev_tear, prev_new,
                        );
                    } else {
                        tear_guesses = new_tear_streams.clone();
                    }
                    prev_tear_values = Some(tear_value_snapshot);
                    prev_new_values = Some(new_tear_values);
                },
                ConvergenceMethod::Newton => {
                    tear_guesses = new_tear_streams;
                },
            }
        }

        Err(format!("Failed to converge after {} iterations",
            self.max_iterations).into())
    }

    fn calculate_residual(&self, old_streams: &HashMap<String, MaterialStream>,
        new_streams: &HashMap<String, MaterialStream>) -> f64 {
        let mut sum_sq = 0.0;
        for tear_id in &self.tear_streams {
            if let (Some(old), Some(new)) = (old_streams.get(tear_id), new_streams.get(tear_id)) {
                let delta_fiber = (new.fiber_flow - old.fiber_flow) / old.fiber_flow.max(1.0);
                let delta_water = (new.water_flow - old.water_flow) / old.water_flow.max(1.0);
                sum_sq += delta_fiber.powi(2) + delta_water.powi(2);
            }
        }
        sum_sq.sqrt()
    }

    fn wegstein_update(&self, old_streams: &HashMap<String, MaterialStream>,
        new_streams: &HashMap<String, MaterialStream>,
        prev_old: &HashMap<String, f64>,
        prev_new: &HashMap<String, f64>) -> HashMap<String, MaterialStream> {
        let mut updated = HashMap::new();

        for tear_id in &self.tear_streams {
            if let Some(stream_old) = old_streams.get(tear_id) {
                if let Some(stream_new) = new_streams.get(tear_id) {
                    let x_k = stream_old.fiber_flow;
                    let x_new = stream_new.fiber_flow;

                    let q = if let (Some(&x_k_prev), Some(&x_new_prev)) =
                        (prev_old.get(tear_id), prev_new.get(tear_id)) {
                        let s = (x_new - x_new_prev) / (x_k - x_k_prev + 1e-10);
                        let q_raw = s / (s - 1.0);
                        q_raw.clamp(-5.0, 0.5)
                    } else {
                        0.5
                    };

                    let x_next = x_k + q * (x_new - x_k);

                    let mut updated_stream = stream_new.clone();
                    let scale = x_next / x_new.max(1e-6);
                    updated_stream.fiber_flow = x_next;
                    updated_stream.water_flow *= scale;

                    updated.insert(tear_id.clone(), updated_stream);
                }
            }
        }

        updated
    }
}
```

---

## Phase Breakdown

### Phase 1 — Scaffold + Auth + Database (Days 1–4)

**Day 1: Project initialization**
- `cargo init papermill-api`
- Add dependencies: axum, tokio, sqlx, serde, uuid, chrono, aws-sdk-s3, redis
- `src/main.rs` — Axum app with CORS, tracing, graceful shutdown
- `src/config.rs` — Environment config
- `src/state.rs` — AppState (PgPool, Redis, S3Client)
- `docker-compose.yml` — PostgreSQL, Redis, MinIO

**Day 2: Database schema**
- `migrations/001_initial.sql` — All tables
- `src/db/models.rs` — SQLx structs
- Run migrations
- Seed fiber properties and chemicals

**Day 3: Authentication**
- `src/auth/mod.rs` — JWT middleware
- `src/auth/oauth.rs` — Google/GitHub OAuth
- `src/api/handlers/auth.rs` — Register, login, OAuth callback
- Password hashing with bcrypt

**Day 4: User and flowsheet CRUD**
- `src/api/handlers/users.rs` — Profile CRUD
- `src/api/handlers/flowsheets.rs` — Create, list, get, update, delete, fork
- `src/api/handlers/orgs.rs` — Org management
- Integration tests

### Phase 2 — Solver Core (Days 5–12)

**Day 5: Mass/energy balance framework**
- `solver/src/streams.rs` — MaterialStream struct
- `solver/src/units/mod.rs` — UnitOperation trait
- `solver/src/species.rs` — Chemical species database
- Unit tests

**Day 6: Basic unit operations**
- `solver/src/units/mixer.rs`
- `solver/src/units/splitter.rs`
- `solver/src/units/heater.rs`
- `solver/src/units/stock_chest.rs`
- Tests with simple mass balance

**Day 7: Kraft digester model**
- `solver/src/units/digester.rs` — Full Purdue kinetics
- H-factor integration
- Kappa number prediction
- Yield calculation
- Validation against mill data

**Day 8: Washing unit operations**
- `solver/src/units/washer.rs` — Drum washer, diffusion washer
- Norden efficiency model
- Dilution factor optimization
- Tests with washing curves

**Day 9: Bleach plant models**
- `solver/src/units/bleach_tower.rs` — D, E, O, P stages
- ClO2, O2, H2O2 kinetics
- Brightness and viscosity prediction
- AOX tracking

**Day 10: Paper machine headbox and wire**
- `solver/src/units/headbox.rs`
- `solver/src/units/wire_section.rs` — Kozeny-Carman drainage
- Consistency profile prediction
- Tests with drainage curves

**Day 11: Paper machine press and dryer**
- `solver/src/units/press_section.rs` — Carlsson consolidation
- `solver/src/units/dryer_section.rs` — Heat/mass transfer
- Moisture profile prediction
- Tests

**Day 12: Sequential modular solver**
- `solver/src/sequential_modular.rs` — Full SM solver
- Tear stream detection
- Wegstein and Newton convergence
- Convergence monitoring

### Phase 3 — Frontend Visualization (Days 13–18)

**Day 13: Frontend scaffold**
- `npm create vite@latest frontend -- --template react-ts`
- Add: zustand, @tanstack/react-query, axios, d3, dagre
- `src/App.tsx` — Router, layout
- `src/stores/flowsheetStore.ts`

**Day 14: Flowsheet editor — canvas**
- `src/components/FlowsheetEditor/Canvas.tsx` — SVG canvas with pan/zoom
- `src/components/FlowsheetEditor/Grid.tsx`
- `src/components/FlowsheetEditor/UnitPalette.tsx` — Unit library sidebar
- Drag-and-drop placement

**Day 15: Flowsheet editor — units and streams**
- `src/components/FlowsheetEditor/UnitSymbol.tsx` — SVG unit renderers
- `src/components/FlowsheetEditor/Stream.tsx` — Stream drawing tool
- Property panel for unit parameters
- Auto-layout with Dagre.js

**Day 16: Stream table viewer**
- `src/components/Results/StreamTable.tsx` — Tabular display
- Filterable columns (fiber, water, species, temperature)
- Export to Excel
- Pagination for large flowsheets

**Day 17: Sankey diagram**
- `src/components/Results/SankeyDiagram.tsx` — D3.js Sankey
- Interactive node/link hover
- Species drill-down (show only fiber, only NaOH)
- Color coding by stream type

**Day 18: Property plots**
- `src/components/Results/PropertyPlot.tsx` — D3.js line charts
- Kappa vs. H-factor
- Moisture vs. dryer position
- Brightness vs. ClO2 charge
- Interactive cursors

### Phase 4 — API + Job Orchestration (Days 19–24)

**Day 19: Simulation API**
- `src/api/handlers/simulation.rs` — Create, get, list, cancel
- Flowsheet validation
- Plan limits enforcement

**Day 20: Simulation worker**
- `src/workers/simulation_worker.rs` — Redis job consumer
- Call solver
- Progress streaming
- S3 result upload

**Day 21: WebSocket progress**
- `src/api/ws/simulation_progress.rs`
- Frontend: `useSimulationProgress.ts` hook
- Real-time convergence history

**Day 22: Fiber and chemical libraries**
- `src/api/handlers/fibers.rs` — Search, get fiber properties
- `src/api/handlers/chemicals.rs` — Chemical database
- Parametric search

**Day 23: Report generation**
- `src/services/report.rs` — PDF generation (headless Chrome)
- Stream table formatting
- Sankey diagram export
- Excel export

**Day 24: Flowsheet templates**
- Pre-built kraft mill template
- Bleach plant template
- Paper machine template
- Template API endpoints

### Phase 5 — Optimization Engine (Days 25–29)

**Day 25: Python optimization service scaffold**
- `optimization-service/main.py` — FastAPI app
- SciPy integration
- Objective function interface

**Day 26: Chemical charge optimization**
- Minimize ClO2 consumption for target brightness
- EA charge optimization for target kappa
- Multi-objective optimization (cost vs. quality)

**Day 27: Yield maximization**
- Optimize cooking H-factor and EA
- Constrain kappa number
- Pareto frontier visualization

**Day 28: Energy optimization**
- Minimize steam consumption
- Optimize dryer section profile
- Pinch analysis integration

**Day 29: Optimization UI**
- `src/components/Optimization/OptimizationPanel.tsx`
- Set objective and constraints
- Run optimization
- Display results with comparison

### Phase 6 — Billing + Plan Enforcement (Days 30–33)

**Day 30: Stripe integration**
- `src/api/handlers/billing.rs`
- `src/api/handlers/webhooks/stripe.rs`
- Plan mapping

**Day 31: Usage tracking**
- `src/middleware/plan_limits.rs`
- `src/services/usage.rs`
- Usage dashboard

**Day 32: Billing UI**
- `frontend/src/pages/Billing.tsx`
- Plan comparison
- Usage meter
- Upgrade flow

**Day 33: Feature gating**
- Gate optimization behind Mill Engineer plan
- Gate Excel export behind Pro
- Locked feature indicators

### Phase 7 — Validation + Testing (Days 34–38)

**Day 34: Solver validation — kraft cooking**
- Benchmark mill data vs. predictions
- Kappa number accuracy
- Yield accuracy
- H-factor validation

**Day 35: Solver validation — paper machine**
- Drainage profile vs. mill data
- Moisture profile validation
- Property predictions

**Day 36: Convergence testing**
- Large recycle loops
- Difficult tear streams
- Stress tests

**Day 37: Integration testing**
- End-to-end flowsheet simulation
- API tests
- WebSocket tests
- Concurrent simulations

**Day 38: Performance testing**
- Solver benchmarks
- Visualization performance
- Load testing

### Phase 8 — Deployment + Launch (Days 39–42)

**Day 39: Docker and Kubernetes**
- `Dockerfile` — Multi-stage Rust build
- `k8s/` — Manifests
- Health check endpoints

**Day 40: CDN and asset delivery**
- CloudFront distribution
- S3 static hosting
- Asset optimization

**Day 41: Monitoring and polish**
- Prometheus metrics
- Grafana dashboards
- Sentry integration
- UI polish

**Day 42: Launch preparation**
- Security audit
- Documentation
- Landing page
- Production deploy

---

## Critical Files

```
papermill/
├── solver/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── streams.rs
│   │   ├── species.rs
│   │   ├── sequential_modular.rs
│   │   ├── convergence.rs
│   │   ├── units/
│   │   │   ├── mod.rs
│   │   │   ├── digester.rs
│   │   │   ├── washer.rs
│   │   │   ├── bleach_tower.rs
│   │   │   ├── headbox.rs
│   │   │   ├── wire_section.rs
│   │   │   ├── press_section.rs
│   │   │   ├── dryer_section.rs
│   │   │   ├── mixer.rs
│   │   │   ├── splitter.rs
│   │   │   ├── heater.rs
│   │   │   ├── stock_chest.rs
│   │   │   └── fan_pump.rs
│   │   └── validation/
│   │       ├── mill_data.rs
│   │       └── benchmarks.rs
│   └── tests/
│       ├── integration.rs
│       └── convergence.rs
│
├── papermill-api/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── state.rs
│   │   ├── error.rs
│   │   ├── auth/
│   │   │   ├── mod.rs
│   │   │   └── oauth.rs
│   │   ├── api/
│   │   │   ├── router.rs
│   │   │   ├── handlers/
│   │   │   │   ├── auth.rs
│   │   │   │   ├── users.rs
│   │   │   │   ├── flowsheets.rs
│   │   │   │   ├── simulation.rs
│   │   │   │   ├── fibers.rs
│   │   │   │   ├── chemicals.rs
│   │   │   │   ├── billing.rs
│   │   │   │   ├── reports.rs
│   │   │   │   └── webhooks/stripe.rs
│   │   │   └── ws/
│   │   │       └── simulation_progress.rs
│   │   ├── middleware/
│   │   │   └── plan_limits.rs
│   │   ├── services/
│   │   │   ├── usage.rs
│   │   │   ├── report.rs
│   │   │   └── s3.rs
│   │   └── workers/
│   │       └── simulation_worker.rs
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── tests/
│       └── api_integration.rs
│
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── stores/
│   │   │   ├── authStore.ts
│   │   │   ├── flowsheetStore.ts
│   │   │   └── simulationStore.ts
│   │   ├── hooks/
│   │   │   └── useSimulationProgress.ts
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Editor.tsx
│   │   │   ├── Templates.tsx
│   │   │   └── Billing.tsx
│   │   ├── components/
│   │   │   ├── FlowsheetEditor/
│   │   │   │   ├── Canvas.tsx
│   │   │   │   ├── UnitPalette.tsx
│   │   │   │   ├── UnitSymbol.tsx
│   │   │   │   ├── Stream.tsx
│   │   │   │   └── PropertyPanel.tsx
│   │   │   ├── Results/
│   │   │   │   ├── StreamTable.tsx
│   │   │   │   ├── SankeyDiagram.tsx
│   │   │   │   └── PropertyPlot.tsx
│   │   │   └── Optimization/
│   │   │       └── OptimizationPanel.tsx
│   │   └── lib/
│   │       └── api.ts
│
├── optimization-service/
│   ├── requirements.txt
│   ├── main.py
│   ├── optimizers/
│   │   ├── chemical_charge.py
│   │   ├── yield_optimization.py
│   │   └── energy_optimization.py
│   └── Dockerfile
│
├── k8s/
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── redis-deployment.yaml
│   ├── optimization-deployment.yaml
│   └── ingress.yaml
│
└── docker-compose.yml
```

---

## Solver Validation Suite

### Benchmark 1: Kraft Cooking — Softwood (Pine)

**Input:**
- Wood flow: 1000 BDT/d Southern Pine
- EA charge: 18% as NaOH
- Sulfidity: 30%
- H-factor target: 1200
- Liquor-to-wood: 4.0

**Expected:**
- Kappa number: **28-32** (typical unbleached kraft)
- Yield: **48-52%** (softwood kraft range)
- Cooking time at 170°C: **2.5-3.5 hours**

**Tolerance:** Kappa ±2, Yield ±2%, Time ±0.5h

### Benchmark 2: Brownstock Washing — 3-Stage

**Input:**
- Inlet consistency: 12%
- Dilution factor per stage: 2.0
- Norden efficiency: 95%

**Expected:**
- Final consistency: **10-12%**
- Dissolved solids removal: **95-98%**
- NaOH loss: **< 2% of inlet**

**Tolerance:** Consistency ±1%, Removal ±2%

### Benchmark 3: Bleach Plant — D0-EOP-D1

**Input:**
- Inlet kappa: 30
- D0 ClO2 charge: 0.8% on pulp
- EOP: 2.0% NaOH, 0.3% O2
- D1 ClO2 charge: 0.3%

**Expected:**
- Final brightness: **88-90% ISO**
- Final kappa: **< 2**
- Viscosity retention: **> 85%**

**Tolerance:** Brightness ±1%, Kappa ±0.5, Viscosity ±5%

### Benchmark 4: Wire Section Drainage

**Input:**
- Headbox consistency: 1.0%
- Wire length: 20m
- 6 drainage elements (foils + vacuum boxes)

**Expected:**
- Final consistency: **18-22%**
- Drainage profile: monotonic increase
- Water removed: **98-99% of total removal**

**Tolerance:** Final consistency ±2%

### Benchmark 5: Full Kraft Mill Convergence

**Input:**
- 50-unit flowsheet with 8 recycle loops
- Wegstein convergence
- Tolerance: 1e-4

**Expected:**
- Convergence: **< 20 iterations**
- Solve time: **< 30 seconds**
- Mass balance closure: **< 0.1% error**

**Tolerance:** Iterations < 30, Time < 60s

---

## Verification

### Integration Testing Checklist

1. **Auth flow** — Register → login → JWT → authorized calls → refresh → logout
2. **Flowsheet CRUD** — Create → update → save → reload → verify state
3. **Flowsheet editing** — Place units → connect streams → set params → validate
4. **Simulation run** — Small flowsheet (10 units) → converges → results returned
5. **Large simulation** — 50-unit mill → job queued → WebSocket progress → results in S3
6. **Stream table** — Load results → display all streams → export Excel
7. **Sankey diagram** — Render flows → hover tooltips → species drill-down
8. **Property plots** — Load time series → pan/zoom → cursors
9. **Fiber library** — Search "Southern Pine" → results → use in digester
10. **Optimization** — Run ClO2 minimization → converges → results displayed
11. **Plan limits** — Free user → 15-unit flowsheet → blocked with upgrade prompt
12. **Billing** — Subscribe to Pro → webhook → plan updated → limits lifted
13. **Templates** — Select kraft mill → flowsheet loaded → simulate → converges
14. **Concurrent** — 5 users simultaneously running sims → all complete
15. **Error handling** — Non-convergent flowsheet → meaningful error → no crash

### SQL Verification Queries

```sql
-- Simulation throughput
SELECT status, COUNT(*) as count, AVG(wall_time_ms)::int as avg_ms
FROM simulations
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY status;

-- User plan distribution
SELECT plan, COUNT(*) as users
FROM users
GROUP BY plan;

-- Convergence method effectiveness
SELECT convergence_method,
    AVG(iterations_count) as avg_iterations,
    AVG(wall_time_ms)::int as avg_ms
FROM simulations
WHERE status = 'converged'
GROUP BY convergence_method;

-- Popular fiber species
SELECT name, category, COUNT(*) as usage_count
FROM fiber_properties fp
JOIN flowsheets f ON f.flowsheet_data::text LIKE '%' || fp.name || '%'
GROUP BY fp.id
ORDER BY usage_count DESC
LIMIT 10;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Solver: 10-unit flowsheet | < 2s | Server timing |
| Solver: 50-unit kraft mill | < 30s | Server timing, 4 cores |
| Solver: 100-unit mill with recycles | < 90s | Server timing, 8 cores |
| Sankey diagram: 100 streams | 60 FPS | Chrome DevTools |
| Stream table: 200 streams | < 500ms render | React Profiler |
| Property plot: 1000 points | 60 FPS | D3.js performance |
| API: create simulation | < 200ms | p95 latency |
| WebSocket: progress latency | < 100ms | Worker to client |
| Excel export: 200 streams | < 3s | Server-side generation |

---

## Deployment Architecture

### Infrastructure

```
┌─────────────────────────────────────────────────┐
│                  CloudFront CDN                   │
│  ┌─────────────┐  ┌─────────────┐                │
│  │ Frontend SPA │  │ PDF Reports │                │
│  └─────────────┘  └─────────────┘                │
└─────────────────────┬───────────────────────────┘
                      │
          ┌───────────┴───────────┐
          │   AWS ALB (HTTPS)     │
          └───────────┬───────────┘
                      │
    ┌─────────────────┼─────────────────┐
    ▼                 ▼                 ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ API      │  │ API      │  │ API      │
│ (Axum)   │  │ (Axum)   │  │ (Axum)   │
│ Pod ×3   │  │          │  │          │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│PostgreSQL│ │ Redis    │ │ S3       │
│ (RDS)    │ │(ElastiC) │ │          │
└──────────┘ └────┬─────┘ └──────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Worker   │ │ Worker   │ │ Optim    │
│ (Rust)   │ │ (Rust)   │ │ (Python) │
│c7g.2xl   │ │c7g.2xl   │ │c7g.xl    │
└──────────┘ └──────────┘ └──────────┘
```

### Scaling Strategy

- **API servers**: HPA at 70% CPU, min 3, max 10
- **Simulation workers**: HPA on Redis queue depth, min 2, max 20
- **Database**: RDS r6g.xlarge with read replicas
- **Redis**: ElastiCache r6g.large cluster

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Chemical Recovery Cycle | Full evaporator train, recovery boiler, recausticizing, lime kiln with sodium/sulfur balance closure | High |
| Dynamic Simulation | Time-domain simulation for grade changes, startup/shutdown, and control system tuning | High |
| Historian Integration | Import process data from OSIsoft PI, Honeywell, Emerson DCS for model calibration and validation | High |
| AI Quality Prediction | ML models predicting paper tensile, tear, brightness from process conditions with transfer learning | Medium |
| Environmental Compliance | Full effluent treatment modeling (ASB, activated sludge), air emissions tracking, permit compliance reporting | Medium |
| Mechanical Pulping | TMP/CTMP refiner models with specific energy, freeness, and fiber length distribution | Medium |
| Recycled Fiber | OCC/MOW pulping, deinking, contaminant removal, and stickies tracking | Low |
| Real-Time Digital Twin | Live connection to mill DCS for real-time process monitoring and operator guidance | Low |
