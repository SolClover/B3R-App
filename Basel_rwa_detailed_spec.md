# Basel 3.1 Credit RWA Calculator - Detailed Technical Specification

## Executive Summary

This document provides a comprehensive specification for implementing a Basel 3.1 Credit Risk-Weighted Assets (RWA) calculator aligned with PRA requirements. The calculator determines the appropriate regulatory capital requirement for credit exposures through a series of integrated calculation engines that process client information, loan characteristics, and apply sophisticated risk mitigation rules.

---

## 1. SYSTEM ARCHITECTURE OVERVIEW

### 1.1 Core Processing Engines

The Basel 3.1 RWA calculator comprises six primary processing engines:

1. **Rules Engine** - Determines calculation approach and applicable scalars
2. **Credit Risk Mitigation (CRM) Engine** - Processes risk mitigation instruments
3. **LGD Engine** - Determines Loss Given Default
4. **EAD Engine** - Calculates Exposure at Default
5. **IRB Calculation Engine** - Executes IRB approach calculations (FIRB/AIRB)
6. **Standardized Calculation Engine** - Executes standardized approach calculations

Plus supporting engines:
7. **Output Floor Validation Engine** - Enforces 72.5% floor and adjustments
8. **Currency & Valuation Engine** - Handles currency mismatches and revaluations
9. **Regulatory Mapping Engine** - Maps exposures to Basel 3.1 asset classes

### 1.2 Data Flow Architecture

```
INPUTS
  ├── Client Information
  ├── Loan Characteristics
  └── Regulatory Parameters
        ↓
RULES ENGINE → Determines approach & scalars
        ↓
CRM ENGINE → Adjusts PD/LGD/EAD
        ↓
SPECIALIZED ENGINES → LGD, EAD, validation
        ↓
IRB & STANDARDIZED ENGINES → Calculate RWs/K values
        ↓
OUTPUT FLOOR ENGINE → Apply floor & adjustments
        ↓
FINAL RWA OUTPUT
```

---

## 2. INPUT DATA REQUIREMENTS

### 2.1 CLIENT INFORMATION

#### 2.1.1 Borrower Profile
- **Client ID**: Unique identifier
- **Legal Name**: Full registered name
- **Country of Incorporation**: Alpha-2 country code (ISO 3166-1)
- **Borrower Type**: 
  - Central government / Central bank
  - Public sector entities
  - Multilateral development banks
  - Banks / Credit institutions
  - Investment firms
  - Large corporates (revenue >€500m for AIRB eligibility)
  - SMEs (turnover <€50m)
  - Individuals (retail customers)
  - Other
- **Sector Classification**: Per ECB NACE classification or equivalent
- **Sub-sector**: Detailed sector information for specialized lending identification

#### 2.1.2 Credit Rating (if available)
- **Internal PD Rating**: 
  - PD grade / grade bucket
  - Associated PD value (0.03% to 100% range)
  - Confidence level / rating confidence
  - Last assignment date
  - Rating history (for trends)
- **External Ratings** (if applicable):
  - Rating agency: S&P, Moody's, Fitch, Bloomberg, etc.
  - Assigned rating
  - Outlook (stable, positive, negative)
  - Date of assignment
  - Long-term vs short-term rating
- **SCRA Score** (Standardized Credit Risk Assessment, for unrated institutions under SA)

#### 2.1.3 Financial Metrics (for corporates, SMEs, institutions)
- **Total Assets** (in EUR equivalent)
- **Total Revenue / Turnover** (last 12 months)
- **Operating Profits / EBITDA**
- **Leverage Ratio** (Debt/EBITDA)
- **Interest Coverage Ratio**
- **Liquidity Ratios** (Current, Quick)
- **Return on Assets (ROA)**
- **Return on Equity (ROE)**
- **Credit Default Swap (CDS) Spread** (if publicly traded/CDS available)
- **Financial Statement Date**: As of which date the financials apply

#### 2.1.4 Relationship & Engagement Data
- **Customer Relationship Status**: Active, Dormant, Closed
- **Years as Customer**
- **Number of Active Exposures**
- **Total Relationship Value** (sum of all exposures to same counterparty)
- **Customer Risk Assessment Score**
- **KYC / AML Status**: Verified, Enhanced, In Progress
- **Regulatory Status**: Regulated, Unregulated, Government-backed

### 2.2 LOAN/EXPOSURE CHARACTERISTICS

#### 2.2.1 Exposure Basics
- **Exposure ID**: Unique transaction identifier
- **Exposure Type**:
  - On-balance sheet (funded)
  - Off-balance sheet (unfunded commitment)
  - Contingent liability
  - Credit derivative position
- **Originated Date**: When facility was first drawn/committed
- **Current Date**: Valuation date
- **Remaining Maturity** (in years, with floor of 0.25 years)
- **Original Maturity**: At origination

#### 2.2.2 Exposure Classification - Asset Class
Per Basel 3.1 PRA requirements, assign to one of:
- **Central Governments & Central Banks**
- **Institutions** (banks, credit institutions)
  - Note: No AIRB approach available under Basel 3.1
- **Corporates**
  - Large corporates (revenue >€500m) - AIRB possible with restrictions
  - SMEs (turnover €50m) - Retail treatment or corporate treatment
  - Large financial corporates - FIRB only
  - Non-financial corporates
- **Retail** (with sub-classes):
  - Qualifying revolving exposures
  - Home mortgages on residential property
  - Transactor exposures
  - Other retail
- **Equities** (SA only - no IRB approach under Basel 3.1)
  - Listed equities
  - Non-listed equities
  - Venture capital exposures
- **Specialized Lending**
  - Project finance
  - Income-producing real estate (IPRE) - FIRB only with 1.5x multiplier
  - Commercial real estate (CRE) - varies by type
  - Object finance
  - Commodity finance
- **Past Due / Non-Performing** (with sub-classification)

#### 2.2.3 Exposure Amount & Structure
- **Outstanding Amount** (Drawn): In EUR equivalent
- **Undrawn Commitment**: In EUR equivalent
- **Exposure at Default (EAD)** - Used if pre-calculated; otherwise calculated by EAD Engine
- **Credit Limit**: Maximum authorized exposure
- **Revolving Status**: 
  - Fixed maturity
  - Evergreen (no maturity)
  - Revolving credit facility
- **Multiple Currency Exposures**:
  - Currency code (ISO 4217)
  - Conversion rate applied
  - Reference rate (ECB, other)
  - Conversion date

#### 2.2.4 Credit Risk Mitigation (CRM) Instruments
For each CRM instrument on the exposure:

**Collateral Details:**
- **Collateral Type**:
  - Cash or cash equivalents (0% volatility)
  - Government debt (0-100% volatility depending on rating)
  - Corporate bonds investment grade (8-10% volatility)
  - Equities (equity indices) (15% volatility)
  - Residential real estate / property
  - Commercial real estate / property
  - Receivables
  - Equipment / vehicles
  - Commodity (precious metals, other)
  - Other
- **Collateral Value (Market Value)**: In EUR equivalent
- **Valuation Method**: 
  - Professional valuation
  - Automated Valuation Model (AVM)
  - Market quote
  - Other
- **Valuation Date**: When collateral was valued
- **Next Revaluation Date**: For properties, mandatory frequency
- **Volatility Adjustment**: Applied per Basel 3.1 (differs by collateral type and maturity mismatch)
- **Maturity (for securities)**: Days to maturity
- **Maturity Mismatch**: Flag if collateral maturity < exposure maturity
- **Currency of Collateral**: If different from exposure
- **Legal Documentation**: Pledge agreement, security interest created, perfected
- **Legal Right Assessment**: Does bank have immediate right to seize and liquidate?
- **Cross-Currency Basis Adjustment**: If applicable

**Guarantee / Counter-Party Protection Details:**
- **Guarantor / Protection Provider**:
  - Type (bank, insurance company, export credit agency, multinational development bank, etc.)
  - Country of incorporation
  - Rating (if available)
  - PD (if available)
  - Internal rating
- **Guarantee Type**:
  - Direct guarantee
  - Conditional guarantee
  - Partial guarantee
  - First-loss protection (credit derivative)
- **Coverage Percentage**: What % of exposure is covered
- **Coverage Type**: Full coverage, first-loss, or specified amount
- **Guarantee Documentation**: Reference to legal agreement
- **Guarantee Maturity**: In years
- **Right to Set-Off**: Whether applicable

#### 2.2.5 Purpose & Origination Details
- **Loan Purpose**:
  - Working capital / general corporate
  - Equipment acquisition / lease
  - Real estate acquisition / construction
  - Project finance / acquisition
  - Bridge finance
  - Acquisition / M&A
  - Refinancing / debt restructuring
  - Trade finance
  - Other
- **Interest Rate Type**: Fixed, floating, stepped
- **Floating Rate Reference**: LIBOR, SOFR, SONIA, EURIBOR, etc.
- **Spread over Reference**: In basis points
- **Pricing Information**: Internal rate of return, margin vs benchmark
- **Documentation Status**: Legally binding, in process, letter of intent

#### 2.2.6 Performance & Payment Data
- **Current Payment Status**: 
  - Performing (on-time payments)
  - 1-29 days past due
  - 30-89 days past due
  - 90+ days past due
  - In default (formal default event)
  - Cure status (if previously defaulted, has it been cured)
- **Days Past Due**: Current days in arrears
- **Probability of Default (PD)** (AIRB only):
  - 1-year PD
  - Multi-year PD
  - PD floor (per Basel 3.1: 0.03% for corporate/large corp, 0.03% for institutions, varying for other classes)
  - PD methodology / model used
- **Frequency of Realization**: For revolving exposures, utilization pattern
- **Average Facility Utilization** (for commitments)

#### 2.2.7 Collateral & Property-Specific (for Real Estate Exposures)
- **Property Type**:
  - Residential (1-4 family, apartment)
  - Commercial (office, retail, industrial, hospitality)
  - Land / development
  - Mixed-use
  - Self-build / owner-occupied
  - Land acquisition, development, construction (ADC)
- **Loan-to-Value (LTV) Ratio**: As of valuation date
- **Original LTV**: At origination
- **Dependence on Cash Flows**: 
  - Not dependent (majority of repayment from other sources)
  - Dependent (material reliance on property cash flows)
- **Property Location**: Country, region, city
- **Market Conditions**: Appreciation / depreciation trends
- **Percentage of Firm's Total RWA**: For large exposures requiring annual revaluation
- **Property Market Risk Indicator**: Market decline indicator for trigger revaluations

---

## 3. RULES ENGINE SPECIFICATION

### 3.1 Primary Function
The Rules Engine determines:
1. Which calculation approach(es) apply (FIRB, AIRB, or Standardized)
2. Which scalars must be applied
3. Which specialized rules override standard calculations
4. Eligibility for alternative treatments

### 3.2 Rules Decision Logic

#### 3.2.1 Asset Class-Based Routing

**Central Governments & Central Banks:**
- Standardized approach only
- Special risk weights per credit rating (0%, 20%, 50%, 100%)
- No IRB approach permitted

**Institutions (Banks/Credit Institutions):**
- Standardized approach mandatory under Basel 3.1 PRA
- FIRB approach available only for specific institutions (FIs with revenue >€500m)
- No AIRB approach permitted (removed under Basel 3.1)
- Special considerations:
  - Asset Value Correlation (AVC) multiplier (1.25x) if regulated FI with assets >US$100bn
  - Firm Size Adjustment (FSA) not available for institutions under Basel 3.1

**Corporates - Large (Revenue >€500m):**
- AIRB approach permitted with restrictions
- FIRB approach mandatory fallback
- Standardized approach available if permission obtained for roll-out class
- Check: Is this a financial sector entity?
  - If yes → FIRB only (AIRB removed for large FIs under Basel 3.1)
  - If no → AIRB or FIRB available

**Corporates - SMEs (Revenue <€50m):**
- Retail treatment eligible if meets criteria:
  - Small business or self-employed
  - Annual revenue <€50m
  - Portfolio approach or individual assessment
- If qualified for retail: Lower risk weights apply
- If corporate treatment: Standard corporate approach
- Firm Size Adjustment (FSA) can reduce PD floor

**Retail - Qualifying Revolving Exposures:**
- IRB approach only if permitted
- Standardized approach: 75% risk weight
- Special capital requirement calculation: K = (LGD × Φ(R) + R × Φ(PD) - PD) × (1 + (M-2.5) × b)
- Currency mismatch risk factor: 1.08x multiplier if currency mismatch present

**Retail - Home Mortgages:**
- FIRB or Standardized approach
- No AIRB for mortgages
- Risk weights: 20%-105% (standardized, based on LTV)
- Applied multipliers:
  - 1.0 for standard mortgages
  - 1.4 for mortgages on second homes
  - 1.7 for mortgages on non-residential property
  - 2.5 for mortgages on 5+ investment properties
- Maximum LTV-based risk weight floor

**Retail - Transactor Exposures:**
- 45% risk weight (standardized) for qualifying transactors
- FIRB: Standard formula with 45% risk weight application

**Equities:**
- Standardized approach mandatory (no IRB permitted under Basel 3.1)
- Risk weights:
  - Venture capital: 400%
  - Non-listed: 250%
  - Listed: Depends on treatment (equity indices, equity baskets, or lookup)
- 5-year phase-in period for transition from IRB

**Specialized Lending (Project Finance, IPRE, CRE, etc.):**
- Income-Producing Real Estate (IPRE): FIRB only with 1.5x multiplier applied to RW
- Commercial Real Estate (CRE): 
  - Slotting: Supervisory slotting approach (PF-type assets)
  - FIRB with 1.5x multiplier: For CRE not dependent on cash flows
  - Standardized: Risk weights 60%-150% based on LTV
- Project Finance: Slotting or FIRB approach depending on structure

**Past Due / Non-Performing Exposures:**
- Special treatment rules:
  - If 90+ days past due and not fully provided for: 
    - Calculate RWA using normal approach, then apply floor: RWA ≥ 1.08 × (EAD - Provisions)
  - If default event recognized:
    - PD = 100% (or use IRB default assumption)
    - Adjusted LGD calculations may apply

#### 3.2.2 Scalar Application Logic

**Asset Value Correlation (AVC) Multiplier:**
- Rule: Applies to exposures to regulated FIs with total assets >US$100bn (measured in bank's home currency)
- Calculation: Multiply correlation parameter by 1.25
- Classes affected:
  - Institutions (when using AIRB, if applicable)
  - Large corporate financial institutions (when using FIRB)
- Effect: Increases RW by ~5-10% depending on other parameters

**Firm Size Adjustment (FSA):**
- Rule: Reduces PD floor for SME corporate exposures
- PD floor: 0.03% (normal) vs 0.03% (no reduction under Basel 3.1 PRA for SMEs)
- Note: Basel 3.1 PRA removed some FSA benefits vs Basel 2.5
- Still available: SMEs are eligible for lower threshold treatments and retail classification

**Maturity Adjustment (M):**
- Rule: Applied to all corporate and institution exposures (FIRB/AIRB)
- Formula component: (M - 2.5) × b
- Calculation of b: 
  - For corporates: b = (0.11852 - 0.05478 × ln(PD))²
  - For institutions: b = (0.11852 - 0.05478 × ln(PD))²
- Floor: M minimum 1 year, maximum 5 years (values outside this range capped)
- Applied in capital requirement formula

**Currency Mismatch Risk Adjustment:**
- Rule: 1.08x adjustment applied to RWA if exposure currency differs from borrower's income currency
- Applies to: Retail exposures (especially qualifying revolving and transactors)
- Documentation requirement: Must identify mismatch

**Income Producing Real Estate (IPRE) / Commercial Real Estate (CRE) Multiplier:**
- IPRE: 1.5x multiplier applied to risk weight (FIRB approach only)
- CRE dependent on cash flows: Similar treatment
- CRE not dependent on cash flows: 
  - Standardized: Risk weight 60%-150% based on LTV
  - No 100% floor removed under Basel 3.1

**Output Floor Scalar:**
- Rule: Total RWA ≥ 72.5% of Standardized RWA
- Applied at portfolio/entity level as final adjustment

#### 3.2.3 Special Rulings Engine

Certain exposure types trigger special calculation rules:

**"Double Default" Treatment (Removed Basel 3.1):**
- Previously: IRB firms could apply 0% risk weight to a portion of guaranteed exposure
- Current: Not permitted under Basel 3.1 - must use substitution approach for guarantees

**Subordinated Debt / Hybrid Instruments:**
- Treatment: Depends on capital/liability classification
- If Tier 2 capital: Different risk weight calculation
- If Tier 1 capital: Deduction from capital (not included in RWA)

**Short-Maturity Exposures:**
- Maturity < 3 months: Reduced risk weight treatment possible in standardized approach
- IRB approach: Still applies full maturity adjustment

**Trade Finance Exemption:**
- Specific eligible trade finance: 0% risk weight (short-maturity)
- Must meet all BCBS conditions:
  - Self-liquidating underlying (goods)
  - Short tenor (<12 months typical)
  - Documented transaction

**Repo/Reverse Repo & SFT (Securities Financing Transactions):**
- Special CRM treatment using Financial Collateral Comprehensive Method (FCCM)
- May be netted under Master Netting Agreements (MNAs) if conditions met
- Default Fund Contributions: Alternative approach

---

## 4. CREDIT RISK MITIGATION (CRM) ENGINE

### 4.1 Core Functionality

The CRM Engine processes risk mitigation instruments and produces adjusted parameters:
- **Adjusted PD** (for guarantees with substitution approach)
- **Adjusted LGD** (for collateral or guarantees)
- **Adjusted EAD** (for unfunded commitments with additional risk)

### 4.2 CRM Methods Available

#### 4.2.1 Funded Credit Protection (Collateral)

**Foundation Collateral Method (FCM):**
- Applies to: Most collateral types
- Mechanism: 
  - Calculate collateral value after volatility adjustments: CV = MV × (1 - haircut)
  - Reduction to LGD: LGD' = LGD × [(E - CV) / E] if CV > 0
  - Floor: LGD' ≥ LGD × max(0.1, 1 - CV/E)
- Collateral haircuts (volatility adjustments) per Basel 3.1:
  - Cash: 0% (0% haircut = no volatility reduction)
  - Government debt (AAA-AA rated): 0-2%
  - Government debt (A rated): 4-6%
  - Government debt (BBB rated): 10%
  - Corporate bonds IG (AAA-A): 4-8%
  - Corporate bonds IG (BBB): 12%
  - Equities (indices): 15%
  - Equities (non-index): 25%
  - Residential real estate: 15% (loan-dependent)
  - Commercial real estate: 25% (loan-dependent)
  - Other collateral: 50% or supervisory determination
- Additional adjustments:
  - Currency mismatch haircut: Add 8% if collateral currency differs from exposure
  - Maturity mismatch: 
    - If remaining maturity of collateral < 1 year: Add 25%
    - If remaining maturity < minimum duration: Treat as expired

**Financial Collateral Comprehensive Method (FCCM):**
- Applies to: Securities Financing Transactions (repos, reverse repos, etc.)
- Special rules:
  - Can use netting if eligible Master Netting Agreement (MNA) present
  - MNA must meet Basel 3.1 standards: Allow prompt liquidation/set-off
  - Formula: EAD' = max(0, [(E × (1 + he)) - (C × (1 - hc - hfx + hm))])
    - E = exposure value
    - C = collateral value
    - he = haircut on exposure
    - hc = haircut on collateral
    - hfx = FX haircut
    - hm = maturity haircut

**Unfunded Credit Protection (Guarantees / Credit Derivatives):**
- Two approaches available:

**Substitution Approach (Most Common):**
- Mechanism:
  1. Split exposure into covered and uncovered portions based on guarantee amount
  2. For covered portion: Apply guarantor's risk parameters (PD, LGD, correlation)
  3. For uncovered portion: Apply original obligor parameters
  4. Calculate RW and capital for each portion separately
  5. Combine results
- Guarantor eligibility:
  - Sovereigns (0% sovereign PD)
  - Banks / Financial Institutions
  - Insurance companies (rated at least BBB- or equivalent)
  - Export credit agencies
  - Multilateral development banks
  - Other entities with acceptable rating
- Substitution treatment:
  - Use guarantor's PD
  - Use standard LGD for guarantee (applies by protection type)
  - For partial guarantees: Only percentage covered uses guarantor parameters
- Floor: RW with guarantee ≥ RW without guarantee (no negative benefit)

**LGD Adjustment Approach (Alternative):**
- Mechanism:
  - Keep original obligor PD
  - Reduce LGD based on guarantee coverage and guarantor quality
  - LGD' = LGD × (1 - (coverage % × guarantor quality factor))
- Guarantor quality factor: 
  - AAA-AA: 0.95
  - A: 0.90
  - BBB: 0.80
  - Below BBB: 0.75
  - Unrated: 0.70
- Net effect: Comparable to substitution but simpler calculation
- Condition: Not double-counting with collateral reduction

**On-Balance Sheet Netting:**
- Applies to: Offsetting receivables and payables with same counterparty
- Requirement: Legally binding netting agreement
- Effect: Reduces net EAD
- Not typically combined with other CRM methods on same exposure

### 4.3 CRM Prioritization Rules

When multiple CRM instruments present on single exposure:

**Priority Order (Maximum Benefit):**
1. Cash collateral: Treat as 0% volatility first
2. Government securities: Process highest-quality first
3. Corporate bonds/equities: Process in descending rating order
4. Real estate collateral: Apply after liquid collateral
5. Guarantees: Apply to any remaining unprotected amount
6. Credit derivatives: Apply to residual risk if permitted

**Calculation Method (Maximize Benefit):**
- Calculate RW reduction from each CRM instrument
- Select combination that minimizes total RWA
- Example:
  - Option A: Collateral reduces LGD by 15%, resulting in RW = 45%
  - Option B: Guarantee reduces effective PD, resulting in RW = 50%
  - Option C: Combination reduces to RW = 40%
  - Use Option C if legally documented

### 4.4 CRM Output

Output from CRM Engine for each exposure:
- **Adjusted PD** (for substitution approach)
- **Adjusted LGD** (after collateral reduction)
- **Adjusted EAD** (if collateral or netting applied)
- **Uncovered EAD** (portion not covered by CRM)
- **CRM Instruments Applied** (audit trail of which instruments used)
- **CRM Benefit** (RWA reduction achieved by applying CRM)

---

## 5. LGD ENGINE SPECIFICATION

### 5.1 Core Functionality

The LGD Engine determines the Loss Given Default percentage to apply to each exposure, considering:
- Asset class of exposure
- Collateral/guarantee status (post-CRM processing)
- IRB vs Standardized approach
- Floors and minimum values per Basel 3.1

### 5.2 LGD Determination by Asset Class

#### 5.2.1 Institutions (Banks/Credit Institutions)

**FIRB Approach (AIRB removed Basel 3.1):**
- **Unsecured Exposures (No Collateral):**
  - Default LGD: 40%
  - Special cases:
    - Subordinated exposures: 100% LGD
    - Senior unsecured: 40%
- **Collateral-Backed:**
  - Apply Foundation Collateral Method (FCM)
  - Start with 40% baseline, reduce per collateral value
  - Floor: 10% LGD (after adjustment)

**Standardized Approach:**
- Risk weights determine capital directly; LGD implicit (typically 45% by default, but varies by rating)
- For rated institutions: Use SCRA (Standardized Credit Risk Assessment)
  - Risk weights: 20%, 30%, 40%, 50%, 60%, 100% based on rating
  - These incorporate implicit LGD

#### 5.2.2 Corporates

**AIRB Approach (if applicable - not for FIs):**
- **Bank-Estimated LGD Model:**
  - Bank develops internal LGD model using historical default data
  - Outputs LGD estimate for each obligor/facility
  - Subject to regulatory validation
- **Input Requirements:**
  - Historical default rates: 7+ years minimum
  - Recovery data: By collateral type, by seniority
  - Workout data: Time to recovery, recovery percentages
- **LGD Floors (Basel 3.1):**
  - Unsecured or backed by non-financial collateral: 10% floor
  - Backed by financial collateral using FCCM: 0% floor (can be lower)
  - Subordinated debt: 35% floor

**FIRB Approach:**
- **Supervisory LGD Values:**
  - Unsecured corporate exposures: 45% LGD
  - Collateral-backed: 
    - Financial collateral (FCCM): 25% LGD
    - Real estate collateral (residential): 35% LGD
    - Real estate collateral (commercial): 40% LGD
    - Other collateral: 40% LGD
  - Subordinated exposures: 75% LGD
- **Adjustments:**
  - Apply FCM per previous section
  - Multiple collateral types: Use highest LGD applicable

**Standardized Approach:**
- Risk weights: 20% (AAA-A), 50% (BBB-BB), 100% (B), 150% (below B), 250% (unrated non-SME), 75% (SME)
- Implied LGD: 45% (standard floor), varies by rating

#### 5.2.3 Retail Mortgages

**FIRB/IRB Approach:**
- **Risk Weight Formula Applied (Post-2008):**
  - Include LGD as parameter: R = 0.15 × Φ(N)^(-1)(PD) + √(0.85) × G
  - But supervisory LGD values apply:
    - Residential mortgages: 20% LGD (senior position)
    - Second homes: 30% LGD
    - Buy-to-let / investment properties: 35% LGD
- **LTV Dependency:**
  - Risk weights vary by LTV ratio
  - Maximum LGD 20% for most residential (senior charge)
  - Increase LGD for junior liens or piggyback mortgages

**Standardized Approach (Basel 3.1):**
- **Risk Weights by LTV:**
  - LTV ≤50%: 20% risk weight
  - LTV ≤60%: 30% risk weight
  - LTV ≤80%: 40% risk weight
  - LTV ≤100%: 50% risk weight
  - LTV >100%: 100% risk weight
- **Multipliers Applied:**
  - Standard mortgages: 1.0x
  - Second homes: 1.4x
  - Non-residential property: 1.7x
  - 5+ investment properties: 2.5x

#### 5.2.4 Retail - Qualifying Revolving & Transactor

**FIRB/IRB Approach:**
- **Supervisory LGD: 75%** for all qualifying revolving
- **Transactor LGD: 35%** (lower due to regular payment activity)
- Collateral consideration:
  - If collateral present: Reduce LGD per FCM method
  - Apply volatility adjustments

**Standardized Approach:**
- Risk weight: 75% (revolving), 45% (transactor)
- Implied LGD: 45% standard

#### 5.2.5 Specialized Lending

**Project Finance:**
- Slotting approach (if using IRB):
  - Assign to supervisory slot: Excellent/Good/Satisfactory/Weak
  - Associated risk weights: 50%/100%/250%/Prohibitive
  - These incorporate implicit LGD
- FIRB approach: 40% LGD (typical) with collateral adjustments

**Income-Producing Real Estate (IPRE):**
- FIRB only (AIRB removed Basel 3.1)
- Supervisory LGD: 40% (senior position)
- Apply 1.5x multiplier to risk weight
- Collateral consideration: Building value; note that IPRE is dependent on cash flows

**Commercial Real Estate (CRE):**
- **If dependent on cash flows (income-producing):**
  - Treat as IPRE: FIRB with 1.5x multiplier
  - LGD: 40%
- **If not dependent on cash flows:**
  - Standardized approach: Risk weights 60%-150% per LTV, no 100% floor
  - IRB approach: LGD 40%, applies RW formula
- **By LTV:**
  - LTV ≤50%: Lower risk weight
  - LTV ≤80%: Standard
  - LTV >80%: Elevated risk weight

#### 5.2.6 Past Due / Non-Performing

**Special LGD Treatment:**
- If 90+ days past due with partial provision:
  - LGD = (EAD - Provision) / EAD (use adjusted effective LGD)
  - Or use standard LGD, apply 1.08x floor to RWA
- If in formal default:
  - IRB: PD = 100%, use default LGD framework
  - Standardized: Often 150% risk weight or prohibitive

### 5.3 LGD Output

LGD Engine output for each exposure:
- **Applied LGD %**: The percentage used for RWA calculation
- **LGD Floor**: Whether floor constraint is binding
- **Collateral Adjustment**: How much LGD was reduced
- **LGD Methodology**: Which method was applied (FCM, supervisory, modeled)

---

## 6. EAD ENGINE SPECIFICATION

### 6.1 Core Functionality

The EAD Engine calculates the Exposure at Default for each exposure, converting contractual amounts into regulatory risk measures.

### 6.2 EAD Calculation by Exposure Type

#### 6.2.1 On-Balance Sheet Funded Exposures

**Simple Calculation:**
- EAD = Current outstanding balance
- Adjustments:
  - Less: Eligible provisions (specific allowances)
  - Plus: Accrued interest (if not yet received)
  - Currency: Convert to reporting currency at spot rate

**Example:**
- Loan balance: EUR 1,000,000
- Specific provisions: EUR 50,000
- EAD = EUR 950,000

#### 6.2.2 Off-Balance Sheet Exposures (Unfunded Commitments)

**Credit Conversion Factor (CCF) Approach:**
- EAD = Undrawn commitment × CCF
- CCF varies by exposure type:

**Standardized Approach CCF:**
- **Commitments with original maturity > 1 year:**
  - General: 20% CCF
  - Corporates, institutions, and central governments: 20%
  - Retail: 20%
  - SME retail: 20%
- **Commitments with original maturity ≤ 1 year:**
  - General: 0% CCF (not included in RWA)
  - Exception: Commitments that are unconditionally cancellable: 0% CCF
  - Exception: Trade-related contingent liabilities: 20% CCF
- **Revolving Commitments (e.g., credit cards, credit lines):**
  - Retail: 20% CCF
  - Corporate: 20% CCF

**IRB Approach - EAD Modeling:**
- **Bank-Estimated Usage Rate (for unfunded commitments):**
  - Based on historical utilization data
  - Input to RWA formula
  - Example: Credit line rarely drawn above 50% → 50% is default usage
- **CCF = Usage Rate:**
  - Minimum floor: 0% (commitment is truly contingent)
  - Bank models expected utilization at time of default
- **Input Requirements:**
  - Historical utilization data: 7+ years
  - By product type, by obligor behavior
  - By economic cycle

**Formula (for IRB)::**
- EAD = Drawn + (Undrawn × CCFIRB)
- CCFIRB = Bank-estimated or supervisory floor (typically 50-100% depending on asset class)

#### 6.2.3 Derivative Exposures & Securities Financing Transactions

**Current Exposure Method (CEM):**
- Rarely used for credit risk RWA (mainly for CVA); included for completeness
- EAD = CE + PFE × multiplier
- Where: CE = current mark-to-market value; PFE = potential future exposure

**Standardized Credit Risk Approach (Securitization):**
- Special EAD treatment for securitized exposures
- Not typically part of standard credit risk RWA (covered under securitization framework)

#### 6.2.4 Special EAD Adjustments

**Default Fund Contributions:**
- For central counterparties (CCPs):
  - Contribution to default fund: Generally 2% risk weight
  - EAD = Actual contribution amount

**Maturity Adjustments (IRB only):**
- EAD is adjusted based on remaining maturity in capital requirement formula
- Does not change EAD value, but affects K calculation

**Undrawn Commitment Exclusion:**
- If commitment is unconditionally cancellable by bank on short notice:
  - EAD includes 0% CCF (no conversion)
  - Must document right to cancel

#### 6.2.5 Currency Conversion & Reporting

**Multi-Currency Exposures:**
- Convert all exposures to base currency (EUR):
  - Exchange rate: Spot rate at calculation date
  - Alternative: Month-end average rate or mid-rate
  - FX re-evaluation: Minimum quarterly
- FX volatility haircut: Applied to collateral if collateral in different currency

### 6.3 EAD Output

Output from EAD Engine:
- **Final EAD**: Amount used in RWA calculation
- **Drawn Amount**: Current outstanding
- **CCF Applied**: Percentage of undrawn commitment converted
- **Adjustments**: Any provisions, accruals, or currency conversions
- **EAD Floor**: Whether minimum EAD constraint applies

---

## 7. IRB CALCULATION ENGINE

### 7.1 Core Functionality

The IRB Calculation Engine computes risk weights and capital requirements using the Foundation IRB (FIRB) and Advanced IRB (AIRB) approaches per Basel 3.1 PRA specifications.

### 7.2 FIRB Approach

**Applies to:**
- All institutions (banks) - AIRB removed
- Some corporates (large FI-type corporates)
- Most real estate exposures (IPRE, CRE)
- Selected specialized lending

**Capital Requirement Formula:**

For **Corporates, Institutions, Large Financial Entities:**

```
K = [LGD × N((√R × G(PD) + √(1-R) × G(U)) / √(1-R)) - PD × LGD] × (1 + (M - 2.5) × b)
```

Where:
- **K** = Capital requirement (typically 8% × RW, but K is the raw requirement before scaling)
- **LGD** = Loss Given Default (supervisory value)
- **PD** = Probability of Default (bank-estimated, subject to floor)
- **R** = Correlation = 0.12 × (1 - EXP(-50 × PD)) / (1 - EXP(-50)) + 0.24 × (1 - (1 - EXP(-50 × PD)) / (1 - EXP(-50)))
  - Simplified: Correlation function of PD, ranges 0.12-0.24
  - For institutions: Correlation fixed per regulation
  - For large corporates: Correlation varies with PD
  - For financial entities >€500m revenue: R × 1.25 (AVC multiplier)
- **G(x)** = Inverse normal cumulative distribution function (Φ^(-1)(x))
- **N(x)** = Cumulative normal distribution function
- **M** = Maturity (in years, floor 1 year, ceiling 5 years)
- **b** = (0.11852 - 0.05478 × ln(PD))^2
- **U** = Random draw (uniform distribution)

**Simplified Approach (Lookup Tables):**
Many banks use pre-calculated lookup tables mapping (PD, LGD, Maturity) → K values

**Risk Weight Conversion:**
```
RW = K × 12.5  (since capital requirement is K ÷ 12.5 = 8%)
RWA = RW × EAD
```

**For Retail Exposures (Qualifying Revolving):**

```
K = [LGD × N((√R × G(PD) + √(1-R) × G(U)) / √(1-R)) - PD × LGD]
```
- **M** = Not applied for retail (no maturity adjustment)
- **LGD** = Supervisory 75%
- **Correlation** = Function of PD, but simpler formula

**For Retail Mortgages:**

```
K = [LGD × N((√R × G(PD) + √(1-R) × G(U)) / √(1-R)) - PD × LGD]
```
- **R** = 0.15 × correlation parameter
- **LGD** = Supervisory values (20%, 30%, 35%)

**PD Floors (Basel 3.1 PRA):**
- Corporates (general): 0.03% floor
- Corporates (SME): 0.03% floor (FSA benefit removed in Basel 3.1 PRA)
- Institutions: 0.03% floor
- Retail mortgages: 0.03% floor
- Retail other: 0.03% floor

**LGD Floors (Basel 3.1):**
- Unsecured or non-financial collateral: 10% floor
- Financial collateral (FCCM): 0% floor (can go to zero)
- Subordinated: 35% floor

**Output from FIRB:**
- RW (%) for the exposure
- K value (raw capital requirement)
- RWA = RW × EAD

### 7.3 AIRB Approach

**Note: Severely Restricted under Basel 3.1 PRA**
- Available for: Large corporates (revenue >€500m) with specific approvals only
- NOT available for: Institutions, FI corporates, small corporates
- Capital requirement formula: Same as FIRB but with bank-estimated LGD

**Bank-Estimated Parameters (AIRB):**

- **PD (Probability of Default):**
  - Derived from bank's internal rating models
  - Based on 7+ years historical data
  - Subject to floors (0.03% typical)
  - Continuous estimation for each obligor
  
- **LGD (Loss Given Default):**
  - Bank estimates based on recovery data
  - Recovery time, recovery percentage, by collateral type
  - Empirical approach vs. recovery facility approach
  - Subject to floors: 10% (unsecured), 0% (FCCM collateral), 35% (subordinated)
  
- **EAD (Exposure at Default):**
  - Bank estimates default-time exposure
  - For commitments: Estimated CCF
  - Empirically derived

**Validation Requirements (for AIRB):**
- Supervisory approval of model approach
- Backtesting: Estimated PD vs actual default rates
- Stress testing: Model performance under adverse scenarios
- Regulatory report: Annual IRB validation filings

**Output from AIRB:**
- RW (%) for the exposure
- RWA = RW × EAD
- Regulatory premium: No benefit vs FIRB (floor applies, as below)

### 7.4 Multipliers & Adjustments (Applied in Both FIRB & AIRB)

**Asset Value Correlation (AVC) - Multiplier 1.25x:**
- Applied to: Regulated FIs with total assets >US$100bn
- Effect: Increases correlation parameter R by 25%
- Results in: +5-10% RW increase

**Maturity Adjustment (Already in formulas):**
- Applied to: All corporates and institutions
- Does not apply to: Retail

**Specialized Lending Multiplier (1.5x):**
- Applied to: Income-Producing Real Estate (IPRE), certain CRE
- Effect: Multiplies final RW by 1.5
- Example: Calculated RW = 60% → Applied RW = 90%

**Currency Mismatch Multiplier (1.08x):**
- Applied to: Retail exposures with currency mismatch
- Effect: Increases RWA by 8%

---

## 8. STANDARDIZED CALCULATION ENGINE

### 8.1 Core Functionality

The Standardized Engine calculates RWA for all exposures not using IRB approaches, and serves as the floor for IRB banks (output floor of 72.5%).

### 8.2 Risk Weight Assignment by Asset Class

#### 8.2.1 Central Governments & Central Banks

**Risk Weights by Rating:**
- AAA to AA-: 0% risk weight
- A+ to A-: 20% risk weight
- BBB+ to BBB-: 50% risk weight
- BB+ to B-: 100% risk weight
- Below B-: 150% risk weight
- Unrated: 100% risk weight (default, no CQS mapping)

#### 8.2.2 Institutions (Banks, Credit Institutions)

**Using SCRA (Standardized Credit Risk Assessment) - New Basel 3.1:**
- **For Rated Institutions:**
  - Credit Quality Steps mapped from external ratings
  - Risk weights: 20% (CQS1/AAA-AA), 30% (CQS2/A), 40% (CQS3/BBB), 50% (CQS4/BB), 100% (CQS5/B), 150% (CQS6/<B)
  
- **For Unrated Institutions:**
  - SCRA approach applied
  - Risk weight: 30% (if meets specific size/stability criteria), 50% (default unrated)

**Special Cases:**
- Short-term exposure to bank: 20% risk weight
- Interbank exposure subject to 3-month maturity: 20%

#### 8.2.3 Corporates

**Rated Corporates (using External Credit Rating):**
- CQS1 (AAA-AA): 20% risk weight
- CQS2 (A): 50% risk weight
- CQS3 (BBB): 75% risk weight
- CQS4 (BB): 100% risk weight
- CQS5 (B): 150% risk weight
- CQS6 (<B): 150% risk weight

**Unrated Corporates:**
- Investment Grade (non-rated, but assessable as such): 65% risk weight
- Non-investment grade: 135% risk weight

**SME Corporates:**
- SME definition: €50m annual turnover threshold
- Risk weights:
  - Retail SME qualifying: 45% risk weight (reduced from 85%)
  - Non-qualifying SME: 75% risk weight
  - Unrated SME: 75% risk weight
  - SME transactor: 45% risk weight

#### 8.2.4 Retail Exposures

**Qualifying Revolving Exposures:**
- Risk weight: 75%

**Home Mortgages - Risk-Sensitive Approach:**
- By Loan-to-Value (LTV) ratio:
  - LTV ≤ 50%: 20% risk weight
  - LTV ≤ 60%: 30% risk weight
  - LTV ≤ 80%: 40% risk weight
  - LTV ≤ 100%: 50% risk weight
  - LTV > 100%: 100% risk weight
- **Multipliers:**
  - Standard residential mortgages: 1.0x
  - Second homes: 1.4x
  - Non-residential property: 1.7x
  - 5+ investment properties: 2.5x
- **Final RW = Base RW × Multiplier**

**Other Retail Transactor Exposures:**
- Risk weight: 45%

**Other Retail Exposures (not above):**
- Risk weight: 75%

**Retail Exposures with Currency Mismatch:**
- Increase RW by multiplying RWA by 1.08

#### 8.2.5 Equity Exposures

**Listed Equities (Standardized):**
- Generic weighting: 250% risk weight
- Alternative: Equity index methodology (banks can map to indices)

**Non-Listed Equities:**
- Risk weight: 250%

**Venture Capital Exposures:**
- Risk weight: 400%
- Note: 5-year phase-in from IRB, currently transitioning

**Strategic Holdings:**
- If <10% stake: Risk weight 250%
- If ≥10% stake: May be deducted from capital (if material position)

#### 8.2.6 Specialized Lending

**Project Finance:**
- Slotting approach (if PRA permits, else standardized)
- Assigned to slot based on credit quality:
  - Excellent: 50% risk weight
  - Good: 100% risk weight
  - Satisfactory: 250% risk weight
  - Weak: 350% or prohibitive

**Income-Producing Real Estate (IPRE):**
- By LTV:
  - LTV ≤ 55%: 50% risk weight
  - LTV ≤ 70%: 70% risk weight
  - LTV ≤ 85%: 90% risk weight
  - LTV > 85%: 120% risk weight

**Commercial Real Estate (CRE) - Not Income Producing:**
- By LTV:
  - LTV ≤ 50%: 60% risk weight
  - LTV ≤ 60%: 80% risk weight
  - LTV ≤ 80%: 100% risk weight
  - LTV > 80%: 150% risk weight
- Note: No 100% floor removed in Basel 3.1

**Commercial Real Estate (CRE) - Owner-Occupied:**
- Can be treated as corporate exposure (50% typical)
- Or CRE treatment if loan parameters suggest

**Land Acquisition, Development & Construction (ADC):**
- Risk weight: 100% (if LTV ≤ 50%)
- Risk weight: 150% (if LTV > 50%)

**Object Finance (e.g., aircraft, equipment leasing):**
- Risk weight: 100% (general)
- Risk weight: 250% (if LTV >80% or weaker credit quality)

#### 8.2.7 Past Due Exposures

**90+ Days Past Due (Specific Provision <20%):**
- Risk weight: 150%

**90+ Days Past Due (Specific Provision ≥20%):**
- Risk weight: 100%

**90+ Days Past Due (Specific Provision ≥ 20% of unsecured portion):**
- Risk weight: 50%

### 8.3 RWA Calculation (Standardized)

**Simple Formula:**
```
RWA = Risk Weight × EAD
```

**Example:**
- Corporate exposure (rated A)
- EAD = EUR 5,000,000
- Risk weight = 50%
- RWA = 50% × EUR 5,000,000 = EUR 2,500,000

### 8.4 Standardized Output

Output from Standardized Engine:
- RW (%) assigned
- RWA for each exposure
- Basis for RW (rating, LTV, PD status, etc.)

---

## 9. OUTPUT FLOOR VALIDATION ENGINE

### 9.1 Core Functionality & Basel 3.1 PRA Requirements

**Output Floor Rule:**
- Total RWA under IRB approach ≥ 72.5% of Standardized RWA

**Application:**
- Measured at firm level (entire IRB portfolio)
- Calculated annually
- Adjusted for:
  - Difference between expected loss and accounting provisions
  - Transitional arrangements for equity exposures (5-year phase-in)

### 9.2 Floor Calculation

**Adjusted Standardized RWA Formula:**

```
Adjusted Standardized RWA = Standardized RWA + 10% × max(0, EL - Provisions)
```

Where:
- **Standardized RWA**: Total RWA calculated using standardized approach for all exposures
- **EL**: Expected Loss under IRB approach
- **Provisions**: Accounting provisions / loan loss allowances (IFRS 9, etc.)
- **Adjustment**: Converts provision differences to RWA terms

**Output Floor Application:**

```
Final RWA = max(IRB RWA, 0.725 × Adjusted Standardized RWA)
```

### 9.3 Transitional Arrangements

**Equity Exposures (5-Year Phase-In):**
- Year 1: 75% of final floor
- Year 2: 75% of final floor
- Year 3: 80% of final floor
- Year 4: 85% of final floor
- Year 5: 100% of final floor

**IRB Transitional Approach (for qualifying exposures):**
- Some corporates may be grandfathered under transitional rules
- Reduced floor (e.g., 50% in year 1, increasing)
- Documented in PRA approval

### 9.4 Floor Engine Output

- **IRB RWA**: Total RWA from IRB calculations
- **Standardized RWA**: Total RWA if all exposures calculated under standardized
- **Adjusted Standardized RWA**: Post-provision adjustment
- **Floor Amount**: 72.5% × Adjusted Standardized RWA
- **Final RWA**: Maximum of IRB RWA and floor amount
- **Floor Binding**: Flag indicating whether floor constraint is active

---

## 10. SUPPORTING ENGINES

### 10.1 Currency & Valuation Engine

**Functions:**
- Handles multi-currency exposures
- Manages property revaluations
- Applies FX volatility adjustments

**Key Processes:**
- **Daily/Weekly Currency Conversion:**
  - Spot rate from market data feed
  - All exposures converted to EUR (base currency)
  - FX gains/losses recorded for accounting
  
- **Property Revaluation Tracking:**
  - Annual revaluation: For properties >3% of firm RWA
  - Trigger revaluation: If market falls >10%
  - Valuation methods: Professional appraisal, AVM, market data
  - Revaluation frequency: Minimum 5-year cycle
  
- **LTV Recalculation:**
  - LTV = Exposure / Current Property Value
  - Recompute after each revaluation
  - May trigger risk weight adjustment if LTV bracket changes
  
- **Currency Mismatch Identification:**
  - Compare exposure currency vs borrower income currency
  - Flag for 1.08x RWA adjustment (retail)

### 10.2 Regulatory Mapping Engine

**Functions:**
- Maps loan types to Basel 3.1 asset classes
- Determines applicable calculation rules
- Ensures classification accuracy

**Key Classifications:**
- Loan purpose → Specialized lending sub-class
- Borrower type → Asset class
- Maturity profile → Maturity adjustment flags
- Collateral type → Eligible types per regulation

**Conflict Resolution:**
- If exposure fits multiple classifications:
  - Apply most conservative rule
  - Document rationale
  - Escalate if ambiguous

### 10.3 Data Quality & Validation Engine

**Functions:**
- Validates all input parameters
- Flags missing or inconsistent data
- Provides data remediation guidance

**Key Validations:**
- EAD > 0 (cannot calculate RWA for zero exposure)
- PD range: 0.03% to 99.97% (within bounds)
- LGD range: 0% to 100% (sensible values)
- Maturity: 0.25 to 5 years (floors/ceilings applied)
- Collateral value ≤ EAD (if higher, cap at EAD)
- LTV ≥ 0 (cannot be negative)
- Rating dates sensible (not >5 years old typically)

---

## 11. EXAMPLE CALCULATION WALK-THROUGH

### Scenario: Large Corporate Facility with Collateral

**Input Parameters:**
```
Borrower: TechCorp Ltd (large corporate, €750m revenue)
Facility: Term loan, €10m
PD: 1.00%
LGD (before collateral): 45%
Maturity: 3 years
Collateral: Corporate bonds (investment grade), €4m
Collateral volatility: 8%
Approach: Foundation IRB (no AIRB, given size/type)
```

**Step 1: Rules Engine**
- Classification: Large corporate (AIRB possible, but assume FIRB approved)
- Applicable approach: FIRB
- Applicable scalars: AVC check - TechCorp assets €8bn (<€100bn) → no AVC
- Maturity adjustment: Applies (3 years, not at ceiling)
- Specialized lending: Not applicable

**Step 2: CRM Engine**
- Collateral: €4m corporate bonds with 8% volatility
- Haircut: 8% (volatility adjustment)
- Adjusted collateral value: €4m × (1 - 0.08) = €3.68m
- LGD adjustment: LGD' = 45% × [(€10m - €3.68m) / €10m] = 45% × 0.632 = 28.4%
- But floor LGD (non-financial, unsecured portion ≥10%): 28.4% > 10% ✓
- Final LGD: 28.4% (rounded to 28%)

**Step 3: EAD Engine**
- On-balance sheet funded: EAD = €10m (as-is)
- Adjustments: None
- Final EAD: €10m

**Step 4: LGD Engine**
- Output from CRM: LGD = 28.4%
- LGD floor check: 28.4% > 10% ✓
- Applied LGD: 28.4%

**Step 5: IRB Calculation Engine (FIRB)**
- Correlation R = 0.12 × (1 - e^(-50×0.01)) / (1 - e^(-50)) + 0.24 × (1 - (1 - e^(-50×0.01)) / (1 - e^(-50)))
- R ≈ 0.24 (approx for 1% PD)
- Maturity adjustment:
  - b = (0.11852 - 0.05478 × ln(0.01))^2 = (0.11852 - 0.05478 × (-4.605))^2 ≈ 0.047
  - Maturity factor = 1 + (3 - 2.5) × 0.047 = 1.0235
- Capital requirement:
  - K = [28.4% × N((√0.24 × Φ^-1(0.01) + √0.76 × Φ^-1(U)) / √0.76) - 0.01 × 28.4%] × 1.0235
  - Using lookup tables: K ≈ 3.24%
- Risk weight: RW = K × 12.5 = 3.24% × 12.5 = 40.5%

**Step 6: RWA Calculation**
- RWA = 40.5% × €10m = €4,050,000

**Step 7: Standardized Approach (for comparison)**
- Corporate, rated or inferred A: 50% risk weight
- Standardized RWA = 50% × €10m = €5,000,000

**Step 8: Output Floor Check**
- IRB RWA: €4,050,000
- Standardized RWA: €5,000,000
- Floor: 72.5% × €5,000,000 = €3,625,000
- Applied RWA = max(€4,050,000, €3,625,000) = €4,050,000
- Floor not binding

**Final Output:**
```
RWA: €4,050,000
Risk Weight: 40.5%
Regulatory Capital (8%): €324,000
Reason: FIRB large corporate with collateral mitigation
```

---

## 12. ITEMS SUPPLEMENTED TO YOUR ORIGINAL SPECIFICATION

Beyond your initial six engines, the complete implementation requires:

### 12.1 Added Components

1. **Regulatory Mapping Engine**
   - Asset class determination
   - Approach eligibility (AIRB vs FIRB vs Standardized)
   - Conflict resolution

2. **Currency & Valuation Engine**
   - Multi-currency handling
   - Property revaluation workflows
   - LTV updates post-revaluation

3. **Output Floor Validation Engine**
   - Portfolio-level floor calculation
   - Provision adjustment
   - Transitional arrangement tracking

4. **Data Quality & Validation Engine**
   - Parameter bounds checking
   - Consistency validation
   - Data remediation guidance

### 12.2 Added Input Categories

1. **Regulatory Parameters** (not just client/loan info)
   - Risk weight tables
   - PD/LGD/EAD floors
   - Scalar multiplier settings
   - Output floor policy

2. **Financial Metrics** (for credit assessment)
   - Leverage, liquidity, profitability ratios
   - CDS spreads (if available)
   - Financial statement currency & date

3. **Performance Data**
   - Payment status (days past due)
   - Drawdown history (for commitments)
   - Cure history (if previously defaulted)

4. **Valuation & Revaluation Data**
   - Collateral valuation methods
   - Property market indicators
   - Trigger revaluation tracking

5. **Legal & Documentation**
   - Netting agreement presence
   - Security perfection status
   - Guarantee documentation reference

### 12.3 Added Outputs

- CRM benefit quantification
- Floor binding analysis
- Risk weight sensitivity analysis
- Provision vs EL reconciliation
- Audit trail (which rules applied)

---

## 13. KEY BASEL 3.1 PRA CHANGES IMPLEMENTED

1. **Removal of AIRB for Institutions**
   - All banks, credit institutions → FIRB only
   - Improves comparability

2. **Removal of AIRB for Large FI Corporates**
   - Financial corporates (non-bank) → FIRB only
   - Reduces RWA variability

3. **Equity Exposures → Standardized Only**
   - No IRB approach permitted
   - 250% RW (listed), 400% RW (venture capital)
   - 5-year phase-in from IRB

4. **Real Estate Risk-Sensitive Weights**
   - Standardized: Risk weights 20%-105% (residential by LTV)
   - Risk weights 60%-150% (CRE by LTV)
   - Removed 100% floor for certain CRE

5. **SME Treatment Harmonization**
   - Simplified SME definition (€50m turnover)
   - Retail SME treatment more attractive (45% RW)
   - Removed 100% RW floor, adjusted multipliers

6. **Output Floor Implementation**
   - 72.5% of standardized RWA floor
   - Adjusted for provision differences
   - Transitional arrangements for equities

7. **Credit Risk Mitigation Refinement**
   - Foundation Collateral Method (FCM) more detailed
   - Double default removed
   - Eligible guarantor list updated

8. **Maturity Adjustment Limits**
   - Floor: 1 year
   - Ceiling: 5 years
   - Applicable to corporates/institutions only

---

## 14. IMPLEMENTATION RECOMMENDATIONS

### 14.1 Technology Architecture

- **Database Layer**: Store all inputs, parameters, calculation history
- **Rules Engine**: Decision tree / fact-based rule system (e.g., Drools, custom rules engine)
- **Calculation Engines**: Vectorized calculations (pandas, numpy in Python; R in finance teams)
- **API Layer**: Expose calculation engines via REST/gRPC for front-end integration
- **Reporting**: Dashboard for RWA analysis, drill-down capability, audit trail

### 14.2 Data Management

- **Governance**: Clear ownership for each input category
- **Validation**: Automated checks with exception reporting
- **Audit Trail**: Log all calculation decisions, parameter changes
- **Versioning**: Track regulatory rule updates, date effectiveness

### 14.3 Testing Strategy

- **Unit Tests**: Each engine independently
- **Integration Tests**: Cross-engine workflows
- **Regression Tests**: Compare with external benchmarks
- **Scenario Analysis**: Stress test portfolio-level impact

### 14.4 Regulatory Compliance

- **Documentation**: Link each calculation to Basel 3.1 / PRA rule reference
- **Approval Workflows**: Escalation for material parameter changes
- **Reporting Packs**: Automated CB/CBOE submission packs
- **Model Validation**: Annual independent review

---

## CONCLUSION

This specification provides a complete blueprint for implementing a Basel 3.1 Credit RWA calculator aligned with PRA final rules. The architecture separates concerns (Rules → CRM → Parameter Determination → Calculation), supporting both regulatory compliance and business logic flexibility.

Key success factors:
1. **Data quality** - Accurate inputs feed accurate outputs
2. **Regulatory alignment** - Continuously update rules per PRA guidance
3. **Auditability** - Document every decision and parameter
4. **Scalability** - Handle portfolio growth and increasing complexity
5. **Transparency** - Business users understand RW drivers for each exposure

Good luck with your implementation!
