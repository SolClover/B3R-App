# Basel 3.1 RWA Calculator - Detailed Data Flow & Sequencing Diagram

## COMPLETE SYSTEM FLOW ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    PRIMARY INPUT LAYER                                                              │
│                                    (Raw Data Inputs)                                                                │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────────────────────────────────┐
                    │           CLIENT INFORMATION INPUTS                             │
                    ├─────────────────────────────────────────────────────────────────┤
                    │ • Client ID, Legal Name, Country, Sector                        │
                    │ • Borrower Type (Corporate, Bank, SME, Retail, Equity, etc.)    │
                    │ • Internal PD Rating & External Rating (if available)           │
                    │ • Financial Metrics (Assets, Revenue, Leverage, Coverage)       │
                    │ • CDS Spread, Relationship Status, KYC/AML Status              │
                    └─────────────────────────────────────────────────────────────────┘
                                               │
                            ┌──────────────────┼──────────────────┐
                            │                  │                  │
                            ▼                  ▼                  ▼
         ┌──────────────────────────┐    ┌──────────────┐    ┌──────────────┐
         │ LOAN/EXPOSURE INPUTS     │    │ REGULATORY   │    │ REVALUATION  │
         ├──────────────────────────┤    │ PARAMETERS   │    │ DATA (if     │
         │ • Exposure ID, Type      │    ├──────────────┤    │ property)    │
         │ • Originated/Maturity    │    │ • Risk Wgt   │    ├──────────────┤
         │ • Outstanding Amount     │    │   Tables     │    │ • Property   │
         │ • Undrawn Commitment     │    │ • Floors     │    │   Value      │
         │ • Currency               │    │ • Scalars    │    │ • LTV        │
         │ • CRM Instruments        │    │ • Output     │    │ • Valuation  │
         │ • Collateral Details     │    │   Floor Pct  │    │   Method     │
         │ • Guarantee Details      │    │ • CCF Tables │    │ • Market     │
         │ • Interest Rate Terms    │    │ • Haircuts   │    │   Indicators │
         │ • Payment Status         │    │ • Risk Wtgs  │    └──────────────┘
         │ • Days Past Due          │    └──────────────┘
         │ • Purpose & Pricing      │
         │ • LTV, Dependence        │
         │ • Property Location      │
         └──────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         STEP 1: REGULATORY MAPPING ENGINE                                                           │
│                    (Classify & Route - No calculations yet)                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Borrower Type                           │
                            │   • Loan Purpose                            │
                            │   • Sector Classification                   │
                            │   • Maturity Profile                        │
                            │   • Collateral Type                         │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │     REGULATORY MAPPING DECISIONS:               │
                        ├────────────────────────────────────────────────┤
                        │ 1. Assign Asset Class:                          │
                        │    - Central Govt / Central Bank                │
                        │    - Institutions (Banks)                       │
                        │    - Corporates (Large / SME)                   │
                        │    - Retail (Mortgages / Revolving / Transact)  │
                        │    - Equities (Listed / Non-listed / VC)        │
                        │    - Specialized Lending (IPRE / CRE / PF)      │
                        │    - Past Due / Non-Performing                  │
                        │                                                 │
                        │ 2. Determine Applicable Approaches:             │
                        │    - Standardized approach (Always eligible)    │
                        │    - FIRB approach (if asset class permits)     │
                        │    - AIRB approach (Restricted to non-FI large  │
                        │      corporates with approval)                  │
                        │                                                 │
                        │ 3. Flag Specialized Rules:                      │
                        │    - Trade Finance (0% RW)                      │
                        │    - Repos/SFT (FCCM treatment)                 │
                        │    - Default/90+ past due (special treatment)   │
                        │    - Subordinated debt (special LGD)            │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │        OUTPUT FROM MAPPING ENGINE:              │
                        ├────────────────────────────────────────────────┤
                        │ ✓ CLASSIFICATION_ASSET_CLASS                    │
                        │ ✓ CLASSIFICATION_APPROACH_AVAILABLE              │
                        │ ✓ CLASSIFICATION_APPROACH_SELECTED              │
                        │ ✓ CLASSIFICATION_SPECIAL_RULES                  │
                        │ ✓ CLASSIFICATION_SPECIALIZED_LENDING_FLAG       │
                        │ ✓ CLASSIFICATION_PAST_DUE_FLAG                  │
                        └────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                    PARALLEL PATH A: EAD ENGINE (Exposure at Default)                                               │
│                    (Runs in parallel with STEP 2, uses primary inputs only)                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Outstanding Amount (Drawn)              │
                            │   • Undrawn Commitment                      │
                            │   • Exposure Type (Funded/Unfunded)         │
                            │   • Specific Provisions                     │
                            │   • Accrued Interest                        │
                            │   • Currency (for conversion)               │
                            │   • Original Maturity (for CCF)             │
                            │   • Asset Class (from Mapping Engine)       │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │      EAD CALCULATION LOGIC:                     │
                        ├────────────────────────────────────────────────┤
                        │ IF On-Balance Sheet Funded:                     │
                        │   EAD = Outstanding - Provisions + Accrued     │
                        │         [converted to EUR at spot rate]        │
                        │                                                 │
                        │ ELSE IF Off-Balance Sheet (Unfunded Commit):   │
                        │   • Look up CCF from Regulatory Parameters:     │
                        │     - By Maturity: ≤1yr (0%) vs >1yr (20%)      │
                        │     - By Asset Class: Varies                    │
                        │     - By IRB/Standardized: May differ           │
                        │   EAD = Drawn + (Undrawn × CCF)                 │
                        │                                                 │
                        │ ELSE IF Derivative/SFT:                        │
                        │   EAD = CE + PFE [or FCCM for repos]           │
                        │                                                 │
                        │ RESULT: Apply any adjustments                   │
                        │   → Floor: EAD > 0 (cannot be zero)            │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │        OUTPUT FROM EAD ENGINE:                  │
                        ├────────────────────────────────────────────────┤
                        │ ✓ EAD_FINAL [Amount to use in RWA calc]         │
                        │ ✓ EAD_DRAWN [Original drawn amount]             │
                        │ ✓ EAD_CCF_APPLIED [% of commitment converted]   │
                        │ ✓ EAD_ADJUSTMENTS [Provisions, accruals]        │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │         (Flows to downstream step 5)          │
                    │         (IRB & Standardized Engines)          │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                     STEP 2: CURRENCY & VALUATION ENGINE                                                            │
│              (Validates & Normalizes values - Runs parallel with CRM Engine)                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Collateral Currency                     │
                            │   • Exposure Currency                       │
                            │   • Borrower Income Currency                │
                            │   • Collateral Market Value                 │
                            │   • Property Valuation Data (if property)   │
                            │   • Last Valuation Date                     │
                            │   • Revaluation Trigger Data                │
                            │   • Outstanding Amount (for LTV calc)       │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │    CURRENCY & VALUATION LOGIC:                  │
                        ├────────────────────────────────────────────────┤
                        │ 1. Currency Conversion:                         │
                        │    - Spot rate from market feed                 │
                        │    - Convert exposure to EUR (base currency)    │
                        │    - Convert collateral value to EUR            │
                        │                                                 │
                        │ 2. Currency Mismatch Identification:            │
                        │    - IF Collateral Currency ≠ Exposure Cur:     │
                        │      Flag for FX haircut (+8%)                  │
                        │    - IF Borrower Income Cur ≠ Exposure Cur:     │
                        │      Flag for retail currency mismatch 1.08x    │
                        │                                                 │
                        │ 3. Property Revaluation (if applicable):        │
                        │    - Check if revaluation due (annual / trigger)│
                        │    - Calculate LTV = Exposure / Current Value   │
                        │    - Compare to previous LTV for changes        │
                        │    - Note any market indicator triggers         │
                        │                                                 │
                        │ 4. Data Quality Checks:                         │
                        │    - Collateral value ≤ Exposure (cap if > )    │
                        │    - LTV ≥ 0 (cannot be negative)               │
                        │    - Valuation dates reasonable                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │    OUTPUT FROM VALUATION ENGINE:                │
                        ├────────────────────────────────────────────────┤
                        │ ✓ VALUATION_COLLATERAL_EUR [In base currency]   │
                        │ ✓ VALUATION_CURRENCY_MISMATCH_FLAG             │
                        │ ✓ VALUATION_FX_HAIRCUT_APPLICABLE              │
                        │ ✓ VALUATION_RETAIL_MISMATCH_MULTIPLIER [1.08x] │
                        │ ✓ VALUATION_LTV [Current LTV ratio]             │
                        │ ✓ VALUATION_PROPERTY_MARKET_TRIGGER            │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │   (Flows to CRM Engine for collateral adj)    │
                    │   (Flows to downstream for LGD/RW select)     │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                       STEP 3: CRM (CREDIT RISK MITIGATION) ENGINE                                                  │
│         Processes collateral & guarantees → Produces adjusted PD, LGD, EAD                                          │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Collateral Type(s) & Amount(s)          │
                            │   • Collateral Maturity                     │
                            │   • Guarantee Amount & Type                 │
                            │   • Guarantor Details (Rating, PD, Type)    │
                            │   • Coverage Percentage                     │
                            │   • Right to Set-Off                        │
                            │   • Netting Agreement Status                │
                            │                                             │
                            │   INPUT FROM VALUATION ENGINE:              │
                            │   • Collateral Value (EUR)                  │
                            │   • Currency Mismatch Flag & FX Haircut     │
                            │   • LTV Ratio                               │
                            │                                             │
                            │   INPUT FROM REGULATORY PARAMS:             │
                            │   • Collateral Haircuts/Volatility %        │
                            │   • Eligible Guarantor Types                │
                            │   • FCM vs FCCM approach rules              │
                            │                                             │
                            │   INPUT FROM PRIMARY (for calcs):           │
                            │   • Original PD (will be adjusted)          │
                            │   • Original LGD (before mitigation)        │
                            │   • EAD (may be adjusted for netting)       │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │      CRM PROCESSING LOGIC:                      │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ PHASE 1: COLLATERAL PROCESSING (Funded CRM)     │
                        │ ─────────────────────────────────────────────   │
                        │ For each collateral instrument on exposure:     │
                        │                                                 │
                        │ Step 1: Look up haircut from Regulatory Params: │
                        │   Haircut = f(Collateral Type, Rating, Maturity)│
                        │   Examples:                                      │
                        │   - Cash: 0%                                    │
                        │   - Gov Debt (AAA-AA): 0-2%                    │
                        │   - Corp Bonds (IG): 4-12%                     │
                        │   - Equities: 15-25%                           │
                        │   - RE: 15-25%                                 │
                        │                                                 │
                        │ Step 2: Add additional haircuts:                │
                        │   IF Maturity Mismatch (<1yr): Add 25%          │
                        │   IF Currency Mismatch: Add 8%                  │
                        │                                                 │
                        │ Step 3: Calculate adjusted collateral value:    │
                        │   Adjusted_CV = Collateral_EUR × (1 - Haircut) │
                        │                                                 │
                        │ Step 4: Calculate LGD reduction:                │
                        │   Excess_Collateral = Adjusted_CV - 0           │
                        │   Coverage = min(Excess_Collateral / EAD, 100%) │
                        │   LGD_Adjusted = LGD × (1 - Coverage)           │
                        │   OR via FCM: LGD' = LGD × [(E-CV)/E]          │
                        │                                                 │
                        │ Step 5: Apply LGD floor:                        │
                        │   IF Non-financial collateral: LGD_Floor = 10%  │
                        │   IF Financial collateral (FCCM): LGD_Floor = 0%│
                        │   IF Subordinated: LGD_Floor = 35%              │
                        │   LGD_Final = max(LGD_Adjusted, LGD_Floor)      │
                        │                                                 │
                        │ PHASE 2: GUARANTEE PROCESSING (Unfunded CRM)    │
                        │ ───────────────────────────────────────────── │
                        │ For each guarantee/protection on exposure:      │
                        │                                                 │
                        │ Step 1: Validate guarantor eligibility:         │
                        │   - Sovereigns: Eligible (PD ≈ 0%)              │
                        │   - Banks: Eligible (use their PD/Rating)       │
                        │   - Insurance: Eligible if BBB- or better       │
                        │   - ECAs, MDBs: Eligible                        │
                        │   - Others: Case by case                        │
                        │                                                 │
                        │ Step 2: Calculate covered & uncovered portions: │
                        │   Covered_EAD = min(Guarantee_Amt, Total_EAD)   │
                        │   Uncovered_EAD = Total_EAD - Covered_EAD       │
                        │                                                 │
                        │ Step 3: Choose substitution approach (default): │
                        │   For COVERED portion:                          │
                        │   - Use Guarantor's PD (not obligor's PD)       │
                        │   - Use standard LGD for guarantee type         │
                        │   For UNCOVERED portion:                        │
                        │   - Use original obligor PD                     │
                        │   - Use original LGD                            │
                        │   RW_Guarantee = RW_Guarantor_Portion           │
                        │   RW_Obligor = RW_Obligor_Portion               │
                        │   RW_Combined = (RW_G×Covered+RW_O×Uncovered)/E │
                        │                                                 │
                        │ PHASE 3: NETTING (if applicable)                │
                        │ ─────────────────────────────────────────────   │
                        │ IF Master Netting Agreement in place:           │
                        │   AND legally enforceable:                      │
                        │   EAD_Netting = Net(Receivables - Payables)     │
                        │   [Cannot be less than 0]                       │
                        │                                                 │
                        │ PHASE 4: PRIORITIZATION & OPTIMIZATION          │
                        │ ─────────────────────────────────────────────   │
                        │ When multiple CRM instruments present:          │
                        │ 1. Apply cash collateral first (0% haircut)     │
                        │ 2. Apply govt securities next (low haircut)     │
                        │ 3. Apply corporate bonds (medium haircut)       │
                        │ 4. Apply equities/RE (high haircut)             │
                        │ 5. Apply guarantees to remainder                │
                        │ Result: Minimize final RWA                      │
                        │                                                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │        OUTPUT FROM CRM ENGINE:                  │
                        ├────────────────────────────────────────────────┤
                        │ ✓ CRM_ADJUSTED_PD [For substitution approach]   │
                        │ ✓ CRM_ADJUSTED_LGD [After collateral mitigation]│
                        │ ✓ CRM_ADJUSTED_EAD [After netting if applicable]│
                        │ ✓ CRM_UNCOVERED_EAD [Portion without CRM]       │
                        │ ✓ CRM_INSTRUMENTS_APPLIED [Audit trail]         │
                        │ ✓ CRM_BENEFIT_RWA_REDUCTION [Impact]            │
                        │ ✓ CRM_COLLATERAL_HAIRCUTS_APPLIED [Detail]      │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │   (Flows to Rules Engine & LGD/IRB Engines)   │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         STEP 4: RULES ENGINE                                                                       │
│     Determines which calculation approach & which scalars to apply                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PREVIOUS ENGINES:              │
                            │   • CLASSIFICATION_ASSET_CLASS               │
                            │   • CLASSIFICATION_APPROACH_AVAILABLE        │
                            │   • CLASSIFICATION_SPECIAL_RULES             │
                            │                                             │
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Borrower Type & Sector                  │
                            │   • Borrower Assets/Revenue (for FI check)  │
                            │   • Relationship Value (for concentration)  │
                            │   • LTV (if property)                       │
                            │ • Maturity (1-5 year range)                │
                            │                                             │
                            │   INPUT FROM VALUATION ENGINE:              │
                            │   • Currency Mismatch Flag                  │
                            │                                             │
                            │   INPUT FROM REGULATORY PARAMS:             │
                            │   • AVC Threshold (US$100bn)                │
                            │   • FSA Rules                               │
                            │   • Scalar Multiplier Settings              │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │      RULES DETERMINATION LOGIC:                 │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ DECISION 1: SELECT CALCULATION APPROACH         │
                        │ ──────────────────────────────────────────      │
                        │ Based on asset class from Mapping Engine:       │
                        │                                                 │
                        │ IF Asset Class = Central Govt/Central Bank:     │
                        │   → Use STANDARDIZED APPROACH ONLY              │
                        │   → Skip IRB engines                            │
                        │                                                 │
                        │ ELSE IF Asset Class = Institutions:             │
                        │   IF FIRB approved by regulator:                │
                        │     → Use FIRB APPROACH (Basel 3.1 restriction) │
                        │   ELSE                                          │
                        │     → Use STANDARDIZED APPROACH                 │
                        │   Note: AIRB removed under Basel 3.1             │
                        │                                                 │
                        │ ELSE IF Asset Class = Corporates (Large):       │
                        │   IF Borrower is Financial Entity:              │
                        │     → Use FIRB APPROACH ONLY                    │
                        │   ELSE IF AIRB Approved for this obligor:       │
                        │     → Use AIRB APPROACH (if preferred)          │
                        │   ELSE                                          │
                        │     → Use FIRB APPROACH                         │
                        │   FALLBACK: → STANDARDIZED APPROACH             │
                        │                                                 │
                        │ ELSE IF Asset Class = Corporates (SME):         │
                        │   IF Qualifies for Retail treatment:            │
                        │     → Route to RETAIL APPROACH                  │
                        │   ELSE                                          │
                        │     → Use FIRB/STANDARDIZED for corporate       │
                        │                                                 │
                        │ ELSE IF Asset Class = Retail (All variants):    │
                        │   IF IRB Approved:                              │
                        │     → Use FIRB APPROACH                         │
                        │   ELSE                                          │
                        │     → Use STANDARDIZED APPROACH                 │
                        │                                                 │
                        │ ELSE IF Asset Class = Equities:                 │
                        │   → Use STANDARDIZED APPROACH ONLY              │
                        │   → No IRB permitted (Basel 3.1)                │
                        │                                                 │
                        │ ELSE IF Asset Class = Specialized Lending:      │
                        │   IF IPRE (Income-Producing RE):                │
                        │     → Use FIRB APPROACH with 1.5x multiplier    │
                        │   ELSE IF Project Finance:                      │
                        │     → Use SLOTTING or FIRB                      │
                        │   ELSE IF CRE:                                  │
                        │     → Use FIRB (1.5x if cash-flow dependent)    │
                        │       or STANDARDIZED (if not IRB approved)     │
                        │                                                 │
                        │ RESULT: RULES_APPROACH_SELECTED                 │
                        │   (STANDARDIZED / FIRB / AIRB)                  │
                        │                                                 │
                        │ ────────────────────────────────────────────    │
                        │                                                 │
                        │ DECISION 2: IDENTIFY APPLICABLE SCALARS         │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ IF Asset Class = Institutions:                  │
                        │   Check: Total Assets > US$100bn ?              │
                        │   IF YES:                                       │
                        │     → Set SCALAR_AVC_MULTIPLIER = 1.25x        │
                        │     → Effect: Increase correlation by 25%       │
                        │   ELSE                                          │
                        │     → Set SCALAR_AVC_MULTIPLIER = 1.0x         │
                        │                                                 │
                        │ IF Asset Class = Corporates (SME):              │
                        │   Check: FSA eligibility (reduced PD floor)     │
                        │   IF Eligible:                                  │
                        │     → Note FSA applies (0.03% floor remains     │
                        │       in Basel 3.1 PRA, but qualification noted)│
                        │   ELSE                                          │
                        │     → Standard PD floor (0.03%)                 │
                        │                                                 │
                        │ IF RULES_APPROACH_SELECTED = FIRB or AIRB:      │
                        │   Always apply:                                 │
                        │     → SCALAR_MATURITY_ADJUSTMENT = YES          │
                        │     → Formula: (M - 2.5) × b, where M ∈ [1,5]   │
                        │       (Does not apply to retail)                │
                        │                                                 │
                        │ IF Asset Class = IPRE or CRE (cash-flow dep):   │
                        │   Apply:                                        │
                        │     → SCALAR_SPECIALIZED_LENDING_MULT = 1.5x    │
                        │     → Effect: Final RW × 1.5                    │
                        │                                                 │
                        │ IF VALUATION_CURRENCY_MISMATCH_FLAG = TRUE:     │
                        │   (AND Asset Class = Retail)                    │
                        │   Apply:                                        │
                        │     → SCALAR_CURRENCY_MISMATCH_MULT = 1.08x     │
                        │     → Effect: Final RWA × 1.08                  │
                        │                                                 │
                        │ IF CLASSIFICATION_PAST_DUE_FLAG = TRUE:         │
                        │   (AND 90+ days past due)                       │
                        │   Apply:                                        │
                        │     → SCALAR_PAST_DUE_FLOOR = 1.08x             │
                        │     → Formula: RWA ≥ 1.08 × (EAD - Provisions)  │
                        │                                                 │
                        │ RESULT: Build scalar dictionary for downstream  │
                        │ {                                               │
                        │   "AVC": 1.25x or 1.0x,                         │
                        │   "Maturity_Adj": YES/NO,                       │
                        │   "Spec_Lending": 1.5x or 1.0x,                │
                        │   "Currency_Mismatch": 1.08x or 1.0x,           │
                        │   "Past_Due_Floor": 1.08x or 1.0x               │
                        │ }                                               │
                        │                                                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │        OUTPUT FROM RULES ENGINE:                │
                        ├────────────────────────────────────────────────┤
                        │ ✓ RULES_APPROACH_SELECTED                       │
                        │ ✓ RULES_SCALAR_AVC_MULTIPLIER [1.0x or 1.25x]   │
                        │ ✓ RULES_SCALAR_MATURITY_ADJUSTMENT [YES/NO]     │
                        │ ✓ RULES_SCALAR_SPEC_LENDING_MULT [1.0x/1.5x]    │
                        │ ✓ RULES_SCALAR_CURRENCY_MISMATCH [1.0x/1.08x]   │
                        │ ✓ RULES_SCALAR_PAST_DUE_FLOOR [1.0x/1.08x]      │
                        │ ✓ RULES_SPECIAL_TREATMENT_FLAGS                 │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │ (Routes to appropriate calculation engine)    │
                    │ (Downstream engines check RULES_APPROACH_SEL) │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         STEP 5: LGD ENGINE                                                                         │
│         Determines final LGD to use based on asset class & CRM treatment                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PREVIOUS ENGINES:              │
                            │   • CLASSIFICATION_ASSET_CLASS               │
                            │   • RULES_APPROACH_SELECTED                  │
                            │   • CRM_ADJUSTED_LGD [from CRM Engine]       │
                            │                                             │
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Collateral Type (if not in CRM)         │
                            │   • Seniority (Senior vs Subordinated)      │
                            │   • Payment Status (for default)            │
                            │   • Provision Amount (for NPL LGD)          │
                            │                                             │
                            │   INPUT FROM VALUATION ENGINE:              │
                            │   • LTV Ratio                               │
                            │                                             │
                            │   INPUT FROM REGULATORY PARAMS:             │
                            │   • Supervisory LGD Tables                  │
                            │   • LGD Floors per asset class              │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │         LGD DETERMINATION LOGIC:                │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ First check: Has CRM Engine already adjusted?   │
                        │ IF CRM_ADJUSTED_LGD exists:                     │
                        │   → Use CRM_ADJUSTED_LGD as starting point      │
                        │ ELSE                                            │
                        │   → Use base/default LGD for asset class        │
                        │                                                 │
                        │ ─── ROUTE BY APPROACH ──────────────────────    │
                        │                                                 │
                        │ IF RULES_APPROACH_SELECTED = STANDARDIZED:      │
                        │   LGD is implicit in risk weights               │
                        │   → LGD_FINAL = LGD from RW lookup              │
                        │   → Typically 45% standard (varies by rating)   │
                        │                                                 │
                        │ ELSE IF RULES_APPROACH_SELECTED = FIRB:         │
                        │   Use supervisory LGD values by asset class:    │
                        │                                                 │
                        │   IF Asset Class = Institutions:                │
                        │     IF Unsecured: LGD = 40%                     │
                        │     ELSE IF Subordinated: LGD = 100%            │
                        │     ELSE IF Financial Collateral: LGD = 25%     │
                        │     ELSE IF Collateral backed: LGD = 40%        │
                        │     ELSE: LGD = 40%                             │
                        │                                                 │
                        │   IF Asset Class = Corporates:                  │
                        │     IF Subordinated: LGD = 75%                  │
                        │     ELSE IF Unsecured: LGD = 45%                │
                        │     ELSE IF Financial Collateral (FCCM): LGD=25%│
                        │     ELSE IF RE Collateral (residential): LGD=35%│
                        │     ELSE IF RE Collateral (commercial): LGD=40% │
                        │     ELSE IF Other Collateral: LGD = 40%         │
                        │     [After CRM already applied, use adjusted]   │
                        │                                                 │
                        │   IF Asset Class = Retail Mortgages:            │
                        │     IF LTV ≤ 50%: LGD = 20%                     │
                        │     ELSE IF LTV ≤ 70%: LGD = 20-30% (varying)  │
                        │     ELSE IF LTV ≤ 90%: LGD = 30-35%             │
                        │     ELSE IF LTV > 90%: LGD = 35%                │
                        │     Apply Multipliers:                          │
                        │     - Standard: 1.0x                            │
                        │     - Second home: 1.4x                         │
                        │     - Non-residential: 1.7x                     │
                        │     - 5+ investment props: 2.5x                 │
                        │     [Final applies after LTV lookup]            │
                        │                                                 │
                        │   IF Asset Class = Retail (Qualifying Revolving)│
                        │     LGD = 75% (supervisory fixed)               │
                        │     IF Collateral present:                      │
                        │       Reduce via FCM (already done by CRM)      │
                        │                                                 │
                        │   IF Asset Class = Retail Transactor:           │
                        │     LGD = 35% (lower than revolving)            │
                        │                                                 │
                        │   IF Asset Class = Specialized Lending:         │
                        │     IF Project Finance (Slotting):              │
                        │       LGD implicit in slot assignment           │
                        │       → Use risk weight from slot                │
                        │     ELSE IF IPRE:                               │
                        │       LGD = 40% (senior position)               │
                        │     ELSE IF CRE (cash-flow dependent):          │
                        │       LGD = 40%                                 │
                        │     ELSE IF CRE (not cash-flow dependent):      │
                        │       LGD = 40% (FIRB) or implicit (SA)         │
                        │                                                 │
                        │   IF Asset Class = Equities:                    │
                        │     LGD implicit in risk weights (SA only)      │
                        │     → Typically 45% standard                    │
                        │                                                 │
                        │   IF Payment Status = 90+ Days Past Due:        │
                        │     IF Provision < 20% of unsecured:            │
                        │       LGD_Effective = (EAD - Provision) / EAD  │
                        │     ELSE                                        │
                        │       Use standard LGD + 1.08x floor on RWA     │
                        │                                                 │
                        │ ELSE IF RULES_APPROACH_SELECTED = AIRB:         │
                        │   Use bank-estimated LGD model output:          │
                        │   → Input to capital requirement formula        │
                        │   [Already subject to AIRB approval process]    │
                        │                                                 │
                        │ ─── APPLY FLOORS ────────────────────────────   │
                        │                                                 │
                        │ After LGD determination:                        │
                        │                                                 │
                        │ IF Seniority = Subordinated:                    │
                        │   LGD_FLOOR = 35%                               │
                        │   LGD_FINAL = max(LGD, LGD_FLOOR)               │
                        │                                                 │
                        │ ELSE IF Collateral = Non-Financial:            │
                        │   LGD_FLOOR = 10%                               │
                        │   LGD_FINAL = max(LGD, LGD_FLOOR)               │
                        │                                                 │
                        │ ELSE IF Collateral = Financial (FCCM):          │
                        │   LGD_FLOOR = 0%                                │
                        │   LGD_FINAL = max(LGD, LGD_FLOOR)               │
                        │                                                 │
                        │ ELSE                                            │
                        │   LGD_FINAL = LGD (no special floor)            │
                        │                                                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │        OUTPUT FROM LGD ENGINE:                  │
                        ├────────────────────────────────────────────────┤
                        │ ✓ LGD_FINAL [% to use in RWA calculation]       │
                        │ ✓ LGD_FLOOR_ACTIVE [Whether floor applied]      │
                        │ ✓ LGD_COLLATERAL_REDUCTION [Amount of mitigation│
                        │ ✓ LGD_METHODOLOGY [FCM/Supervisory/Modeled]     │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │(Flows to IRB & Standardized Engines for calc) │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                STEP 6A: IRB CALCULATION ENGINE (If FIRB/AIRB selected)                                             │
│      Executes Foundation IRB or Advanced IRB capital requirement formulas                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PREVIOUS ENGINES:              │
                            │   • RULES_APPROACH_SELECTED [FIRB/AIRB]      │
                            │   • CLASSIFICATION_ASSET_CLASS               │
                            │   • LGD_FINAL [from LGD Engine]              │
                            │   • EAD_FINAL [from EAD Engine]              │
                            │   • RULES_SCALAR_MATURITY_ADJUSTMENT         │
                            │   • RULES_SCALAR_AVC_MULTIPLIER              │
                            │   • RULES_SCALAR_SPEC_LENDING_MULT           │
                            │   • CRM_ADJUSTED_PD [if from CRM]            │
                            │   • CRM_ADJUSTED_EAD [if from CRM]           │
                            │                                             │
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Original PD (or from CRM engine)        │
                            │   • Maturity (M in years, range 1-5)        │
                            │   • Payment Status (for PD floor check)     │
                            │                                             │
                            │   INPUT FROM REGULATORY PARAMS:             │
                            │   • PD Floors (by asset class)              │
                            │   • LGD Floors (already applied)            │
                            │   • Correlation lookup table / formula      │
                            │   • Normal distribution tables              │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │      IRB CALCULATION LOGIC:                     │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ PHASE 1: PARAMETER VALIDATION & FLOORING        │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ Validate PD:                                    │
                        │   IF PD < 0.03% (or asset class floor):        │
                        │     PD_FINAL = Floor (e.g., 0.03%)             │
                        │   ELSE IF PD > 99.97%:                         │
                        │     PD_FINAL = 99.97%                          │
                        │   ELSE                                          │
                        │     PD_FINAL = PD (as input or from CRM)        │
                        │                                                 │
                        │ Validate Maturity:                              │
                        │   IF M < 1 year:                                │
                        │     M_FINAL = 1 year                            │
                        │   ELSE IF M > 5 years:                          │
                        │     M_FINAL = 5 years                           │
                        │   ELSE                                          │
                        │     M_FINAL = M                                 │
                        │                                                 │
                        │ (LGD already floored by LGD Engine)             │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 2: CALCULATE CORRELATION (R)              │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ Formula varies by asset class:                  │
                        │                                                 │
                        │ IF Asset Class = Institutions:                  │
                        │   R = 0.12 × (1-EXP(-50×PD))/(1-EXP(-50))      │
                        │       + 0.24 × (1 - (1-EXP(-50×PD))/(1-EXP(-50))|
                        │   Approximation: R ≈ 0.12 to 0.24               │
                        │   (PD-dependent, typically 0.12 for very low PD)│
                        │                                                 │
                        │   IF RULES_SCALAR_AVC_MULTIPLIER = 1.25x:       │
                        │     R_ADJUSTED = R × 1.25                       │
                        │                                                 │
                        │ ELSE IF Asset Class = Corporates (non-FI):      │
                        │   R = 0.12 × (1-EXP(-50×PD))/(1-EXP(-50))       │
                        │       + 0.24 × (1 - (1-EXP(-50×PD))/(1-EXP(-50))│
                        │   Approximation: R ≈ 0.12 to 0.24               │
                        │   (Ranges from ~0.12 for 0.03% PD to            │
                        │    ~0.24 for 30%+ PD)                           │
                        │                                                 │
                        │   IF RULES_SCALAR_AVC_MULTIPLIER = 1.25x:       │
                        │     R_ADJUSTED = R × 1.25                       │
                        │                                                 │
                        │ ELSE IF Asset Class = Retail (all types):       │
                        │   R_BASE = Varies by sub-class:                 │
                        │   - Mortgages: R_BASE = 0.15                    │
                        │   - Qualifying Revolving: R = function(PD)      │
                        │   - Transactor: R = similar to revolving        │
                        │   - Other Retail: R = similar to revolving      │
                        │   (For mortgages: No maturity adjustment)       │
                        │                                                 │
                        │ RESULT: R_FINAL (correlation parameter)         │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 3: CALCULATE MATURITY ADJUSTMENT (b)      │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ Only for non-retail (corporates, institutions): │
                        │                                                 │
                        │ b = (0.11852 - 0.05478 × LN(PD_FINAL))²        │
                        │                                                 │
                        │ Example calculations:                           │
                        │ - If PD = 0.03%: b ≈ 0.078                      │
                        │ - If PD = 0.10%: b ≈ 0.071                      │
                        │ - If PD = 1.00%: b ≈ 0.047                      │
                        │ - If PD = 10.0%: b ≈ 0.025                      │
                        │                                                 │
                        │ RESULT: b_FINAL (maturity adjustment factor)    │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 4: CAPITAL REQUIREMENT FORMULA             │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ FOR CORPORATES / INSTITUTIONS (with maturity):  │
                        │                                                 │
                        │ K = [LGD_FINAL × N((√R_FINAL × Φ^-1(PD) +      │
                        │        √(1-R_FINAL) × Φ^-1(U)) / √(1-R_FINAL))  │
                        │      - PD_FINAL × LGD_FINAL]                    │
                        │     × (1 + (M_FINAL - 2.5) × b_FINAL)           │
                        │                                                 │
                        │ Where:                                          │
                        │   N(x) = Cumulative normal distribution         │
                        │   Φ^-1(x) = Inverse normal (quantile)           │
                        │   U = Standard normal random variable           │
                        │   [In practice, use lookup tables or numerical] │
                        │                                                 │
                        │ FOR RETAIL (mortgages, revolving, transactor):  │
                        │ (No maturity adjustment)                        │
                        │                                                 │
                        │ K = [LGD_FINAL × N((√R_FINAL × Φ^-1(PD) +       │
                        │        √(1-R_FINAL) × Φ^-1(U)) / √(1-R_FINAL))  │
                        │      - PD_FINAL × LGD_FINAL]                    │
                        │                                                 │
                        │ RESULT: K_FINAL (raw capital requirement)       │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 5: CONVERT TO RISK WEIGHT                  │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ RW_IRB = K_FINAL × 12.5                         │
                        │ [Scaling factor: 1/0.08 = 12.5]                 │
                        │                                                 │
                        │ Example:                                        │
                        │   If K = 4%, then RW = 4% × 12.5 = 50%          │
                        │                                                 │
                        │ RESULT: RW_IRB_BASE (before specialized mult)   │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 6: APPLY SPECIALIZED MULTIPLIERS           │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ IF RULES_SCALAR_SPEC_LENDING_MULT = 1.5x:       │
                        │   (For IPRE, CRE cash-flow dependent)           │
                        │   RW_IRB_ADJUSTED = RW_IRB_BASE × 1.5           │
                        │                                                 │
                        │ ELSE                                            │
                        │   RW_IRB_ADJUSTED = RW_IRB_BASE                 │
                        │                                                 │
                        │ RESULT: RW_IRB_FINAL                            │
                        │                                                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │        OUTPUT FROM IRB ENGINE:                  │
                        ├────────────────────────────────────────────────┤
                        │ ✓ IRB_RW [Risk weight %]                        │
                        │ ✓ IRB_K_VALUE [Raw capital requirement]         │
                        │ ✓ IRB_CORRELATION [R parameter used]            │
                        │ ✓ IRB_MATURITY_FACTOR [Maturity adjustment]     │
                        │ ✓ IRB_RWA = IRB_RW × EAD_FINAL                  │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │(Flows to Output Floor Validation Engine)      │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                STEP 6B: STANDARDIZED CALCULATION ENGINE                                                            │
│    (Executes in parallel with 6A if selected as primary or for floor calculation)                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PREVIOUS ENGINES:              │
                            │   • CLASSIFICATION_ASSET_CLASS               │
                            │   • EAD_FINAL [from EAD Engine]              │
                            │   • VALUATION_LTV (if property exposure)     │
                            │                                             │
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • External Rating (if available)          │
                            │   • Risk rating / Grade (if internal)       │
                            │   • Payment Status (for past due rule)      │
                            │   • Provision Amount (for past due LGD)      │
                            │                                             │
                            │   INPUT FROM REGULATORY PARAMS:             │
                            │   • Risk Weight Lookup Tables:              │
                            │     - By Asset Class                        │
                            │     - By Rating / CQS                       │
                            │     - By LTV (for RE)                       │
                            │   • Slotting approach slots (if PF)         │
                            │   • Multipliers (property type, etc.)       │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │   STANDARDIZED RW DETERMINATION LOGIC:          │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ ROUTE BY ASSET CLASS:                           │
                        │                                                 │
                        │ IF Asset Class = Central Govt / Central Bank:   │
                        │   Look up RW by rating:                         │
                        │     AAA-AA-: 0%                                 │
                        │     A+ to A-: 20%                               │
                        │     BBB+ to BBB-: 50%                           │
                        │     BB+ to B-: 100%                             │
                        │     Below B-: 150%                              │
                        │     Unrated: 100%                               │
                        │   RW_SA = Lookup(Rating)                        │
                        │                                                 │
                        │ ELSE IF Asset Class = Institutions:             │
                        │   Use SCRA (Standardized Credit Risk Assessment)│
                        │   IF Rated (CQS mapping from external rating):  │
                        │     CQS1 (AAA-AA): 20% RW                       │
                        │     CQS2 (A): 30% RW                            │
                        │     CQS3 (BBB): 40% RW                          │
                        │     CQS4 (BB): 50% RW                           │
                        │     CQS5 (B): 100% RW                           │
                        │     CQS6 (<B): 150% RW                          │
                        │   ELSE IF Unrated:                              │
                        │     Check: Meets size/stability criteria?       │
                        │     IF Yes: 30% RW                              │
                        │     ELSE: 50% RW                                │
                        │   RW_SA = Lookup(CQS or Unrated status)         │
                        │                                                 │
                        │ ELSE IF Asset Class = Corporates:               │
                        │   IF Rated (external rating available):         │
                        │     CQS1 (AAA-AA): 20% RW                       │
                        │     CQS2 (A): 50% RW                            │
                        │     CQS3 (BBB): 75% RW                          │
                        │     CQS4 (BB): 100% RW                          │
                        │     CQS5 (B): 150% RW                           │
                        │     CQS6 (<B): 150% RW                          │
                        │   ELSE IF Unrated:                              │
                        │     Check: Is this SME?                         │
                        │     IF Yes (€50m turnover threshold):           │
                        │       Check: Qualifies for retail treatment?    │
                        │       IF Yes: 45% RW (retail SME)               │
                        │       ELSE: 75% RW (non-qualifying SME)         │
                        │     ELSE (Large Corp):                          │
                        │       Assess: Investment grade characteristics? │
                        │       IF Yes: 65% RW (IG-like)                  │
                        │       ELSE: 135% RW (non-IG)                    │
                        │   RW_SA = Lookup(Rating/SME/Assessment)         │
                        │                                                 │
                        │ ELSE IF Asset Class = Retail:                   │
                        │   IF Sub-class = Qualifying Revolving:          │
                        │     RW_SA = 75%                                 │
                        │   ELSE IF Sub-class = Mortgages:                │
                        │     Look up RW by LTV:                          │
                        │       LTV ≤ 50%: 20% RW                         │
                        │       LTV ≤ 60%: 30% RW                         │
                        │       LTV ≤ 80%: 40% RW                         │
                        │       LTV ≤ 100%: 50% RW                        │
                        │       LTV > 100%: 100% RW                       │
                        │     RW_SA = Lookup(LTV bracket)                 │
                        │     Apply Multiplier:                           │
                        │       Standard mortgages: 1.0x                  │
                        │       Second homes: 1.4x                        │
                        │       Non-residential prop: 1.7x                │
                        │       5+ investment props: 2.5x                 │
                        │     RW_SA_ADJUSTED = RW_SA × Multiplier         │
                        │   ELSE IF Sub-class = Transactor:               │
                        │     RW_SA = 45%                                 │
                        │   ELSE (Other retail):                          │
                        │     RW_SA = 75%                                 │
                        │   [Apply currency mismatch 1.08x later]         │
                        │                                                 │
                        │ ELSE IF Asset Class = Equities:                 │
                        │   IF Listed Equities:                           │
                        │     Check: Can use equity index approach?       │
                        │     IF Yes: Use index mapping → varies          │
                        │     ELSE: Generic RW = 250%                     │
                        │   ELSE IF Non-listed:                           │
                        │     RW_SA = 250%                                │
                        │   ELSE IF Venture Capital:                      │
                        │     RW_SA = 400%                                │
                        │   [Note: In 5-year phase-in from IRB]           │
                        │                                                 │
                        │ ELSE IF Asset Class = Specialized Lending:      │
                        │   IF Project Finance:                           │
                        │     IF Using Slotting approach:                 │
                        │       Assign to slot: Excellent/Good/Satisf/Weak│
                        │       Excellent: 50% RW                         │
                        │       Good: 100% RW                             │
                        │       Satisfactory: 250% RW                     │
                        │       Weak: 350% or prohibitive                 │
                        │     ELSE (Standardized):                        │
                        │       Use corporate RW with variations          │
                        │                                                 │
                        │   ELSE IF IPRE (Income-Producing RE):           │
                        │     Look up RW by LTV:                          │
                        │       LTV ≤ 55%: 50% RW                         │
                        │       LTV ≤ 70%: 70% RW                         │
                        │       LTV ≤ 85%: 90% RW                         │
                        │       LTV > 85%: 120% RW                        │
                        │     RW_SA = Lookup(LTV bracket)                 │
                        │                                                 │
                        │   ELSE IF CRE (Not Income-Producing):           │
                        │     Look up RW by LTV:                          │
                        │       LTV ≤ 50%: 60% RW                         │
                        │       LTV ≤ 60%: 80% RW                         │
                        │       LTV ≤ 80%: 100% RW                        │
                        │       LTV > 80%: 150% RW                        │
                        │     RW_SA = Lookup(LTV bracket)                 │
                        │     [Note: No 100% floor removed in Basel 3.1]  │
                        │                                                 │
                        │   ELSE IF CRE (Owner-Occupied):                 │
                        │     Option A: Treat as corporate → ~50% RW      │
                        │     Option B: Treat as CRE → use CRE rule       │
                        │                                                 │
                        │   ELSE IF ADC (Land Acq, Dev, Construction):    │
                        │     IF LTV ≤ 50%: RW = 100%                     │
                        │     ELSE: RW = 150%                             │
                        │                                                 │
                        │   ELSE IF Object Finance (Aircraft, Equipment): │
                        │     IF LTV ≤ 80%: RW = 100%                     │
                        │     ELSE: RW = 250%                             │
                        │                                                 │
                        │ ELSE IF Asset Class = Past Due / NPL:           │
                        │   IF 90+ days past due:                         │
                        │     Check: Specific Provision amount            │
                        │     IF Provision < 20% of unsecured portion:    │
                        │       RW_SA = 150%                              │
                        │     ELSE IF Provision ≥ 20% of unsecured:       │
                        │       IF Provision < 20% of total:              │
                        │         RW_SA = 100%                            │
                        │       ELSE (Provision ≥ 20% of total):          │
                        │         RW_SA = 50%                             │
                        │   [Apply 1.08x floor to RWA, not RW]            │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ APPLY CURRENCY MISMATCH (if retail):            │
                        │                                                 │
                        │ IF VALUATION_CURRENCY_MISMATCH_FLAG = TRUE      │
                        │   AND Asset Class = Retail:                     │
                        │   RW_SA_FINAL = RW_SA × 1.08                    │
                        │ ELSE                                            │
                        │   RW_SA_FINAL = RW_SA                           │
                        │                                                 │
                        │ RESULT: RW_SA_FINAL (final standardized RW)     │
                        │                                                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │       OUTPUT FROM STANDARDIZED ENGINE:          │
                        ├────────────────────────────────────────────────┤
                        │ ✓ SA_RW [Risk weight %]                         │
                        │ ✓ SA_RWA = SA_RW × EAD_FINAL                    │
                        │ ✓ SA_RATING_USED [Basis for RW]                 │
                        │ ✓ SA_APPROACH_USED [Direct or slotting, etc.]   │
                        └────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
                    │(Flows to Output Floor Validation Engine)      │
                    │                                               │


┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│              STEP 7: OUTPUT FLOOR VALIDATION ENGINE                                                                │
│    Ensures IRB RWA ≥ 72.5% of Standardized RWA (final output floor)                                                │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                            ┌─────────────────────────────────────────────┐
                            │   INPUT FROM PREVIOUS ENGINES:              │
                            │   • IRB_RWA [from IRB Calculation Engine]    │
                            │   • SA_RWA [from Standardized Engine]        │
                            │   • RULES_APPROACH_SELECTED                  │
                            │   • RULES_SCALAR_CURRENCY_MISMATCH           │
                            │   • RULES_SCALAR_PAST_DUE_FLOOR              │
                            │   • CLASSIFICATION_ASSET_CLASS               │
                            │   • CLASSIFICATION_PAST_DUE_FLAG             │
                            │                                             │
                            │   INPUT FROM PRIMARY LAYER:                 │
                            │   • Expected Loss (EL calculation)          │
                            │   • Provision Amount (IFRS 9 reserves)      │
                            │   • EAD_FINAL                                │
                            │                                             │
                            │   INPUT FROM REGULATORY PARAMS:             │
                            │   • Output Floor Percentage (72.5%)          │
                            │   • Equity Transitional Year (if applicable) │
                            │   • Provision Adjustment Factor (10%)        │
                            └─────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │    OUTPUT FLOOR LOGIC:                          │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ PHASE 1: DETERMINE WHICH FLOOR TO APPLY         │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ IF RULES_APPROACH_SELECTED = STANDARDIZED:      │
                        │   → Skip floor (always using SA)                │
                        │   → FINAL_RWA = SA_RWA                          │
                        │   → FLOOR_APPLIED = FALSE                       │
                        │                                                 │
                        │ ELSE IF RULES_APPROACH_SELECTED = FIRB/AIRB:    │
                        │   → Calculate floor and apply                   │
                        │   → Continue to PHASE 2                         │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 2: CALCULATE ADJUSTED STANDARDIZED RWA    │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ Base Formula:                                   │
                        │ Adjusted_SA_RWA = SA_RWA + (10% × max(0,       │
                        │                            EL - Provisions))   │
                        │                                                 │
                        │ Where:                                          │
                        │   EL = PD × LGD × EAD [Expected Loss]           │
                        │   Provisions = Accounting provisions (IFRS 9)   │
                        │                                                 │
                        │ Example:                                        │
                        │   SA_RWA = €5,000,000                           │
                        │   EL = €100,000 (1% PD × 40% LGD × €250k EAD)  │
                        │   Provisions = €50,000                          │
                        │   Adjustment = 10% × max(0, 100k - 50k)         │
                        │              = 10% × 50,000 = €5,000            │
                        │   Adjusted_SA_RWA = 5,000,000 + 5,000           │
                        │                   = €5,005,000                  │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 3: CALCULATE FLOOR AMOUNT                  │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ Standard Floor:                                 │
                        │ FLOOR_AMT = 0.725 × Adjusted_SA_RWA             │
                        │                                                 │
                        │ Transitional Adjustments (if applicable):       │
                        │   IF Asset Class = Equities AND in 5-yr phase: │
                        │     Year 1-2: FLOOR_AMT = 0.75 × SA_RWA         │
                        │     Year 3: FLOOR_AMT = 0.80 × SA_RWA           │
                        │     Year 4: FLOOR_AMT = 0.85 × SA_RWA           │
                        │     Year 5+: FLOOR_AMT = 1.00 × SA_RWA          │
                        │                                                 │
                        │   IF Corporates under grandfathered transition:  │
                        │     Apply approved reduced floor (PRA documented)│
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 4: SPECIAL CASES & MULTIPLIERS             │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ IF CLASSIFICATION_PAST_DUE_FLAG = TRUE          │
                        │   (AND 90+ days past due):                      │
                        │   SPECIAL_FLOOR = 1.08 × (EAD - Provisions)    │
                        │   [This is a hard floor, different than above]  │
                        │   FLOOR_AMT = max(FLOOR_AMT, SPECIAL_FLOOR)     │
                        │                                                 │
                        │ IF RULES_SCALAR_CURRENCY_MISMATCH = 1.08x:      │
                        │   [Applied to RWA, not to floor itself]         │
                        │   → Note but apply to final result (next phase) │
                        │                                                 │
                        │ ──────────────────────────────────────────      │
                        │ PHASE 5: APPLY FLOOR & MULTIPLIERS               │
                        │ ──────────────────────────────────────────      │
                        │                                                 │
                        │ Combine IRB RWA with floor:                     │
                        │ COMBINED_RWA = max(IRB_RWA, FLOOR_AMT)          │
                        │                                                 │
                        │ Apply currency mismatch multiplier (if retail): │
                        │ IF RULES_SCALAR_CURRENCY_MISMATCH = 1.08x:      │
                        │   FINAL_RWA_WITH_MISMATCH = COMBINED_RWA × 1.08x│
                        │ ELSE                                            │
                        │   FINAL_RWA_WITH_MISMATCH = COMBINED_RWA        │
                        │                                                 │
                        │ Apply past-due floor (if 90+ days past due):    │
                        │ IF CLASSIFICATION_PAST_DUE_FLAG = TRUE:         │
                        │   FINAL_RWA = max(FINAL_RWA_WITH_MISMATCH,      │
                        │                   1.08 × (EAD - Provisions))    │
                        │ ELSE                                            │
                        │   FINAL_RWA = FINAL_RWA_WITH_MISMATCH           │
                        │                                                 │
                        │ RESULT: FINAL_RWA (final risk-weighted assets)  │
                        │                                                 │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼
                        ┌────────────────────────────────────────────────┐
                        │  OUTPUT FROM OUTPUT FLOOR ENGINE:               │
                        ├────────────────────────────────────────────────┤
                        │ ✓ FINAL_RWA [Final RWA output]                  │
                        │ ✓ IRB_RWA [Before floor]                        │
                        │ ✓ SA_RWA [Standardized (for reference)]         │
                        │ ✓ ADJUSTED_SA_RWA [With provision adj]          │
                        │ ✓ FLOOR_BINDING [Flag: is floor active?]        │
                        │ ✓ FLOOR_AMOUNT [The floor RWA value]            │
                        │ ✓ RWA_REDUCTION [IRB benefit vs SA]             │
                        │ ✓ REGULATORY_CAPITAL = FINAL_RWA × 8%           │
                        └────────────────────────────────────────────────┘
                                            │
                                            ▼

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                        FINAL OUTPUT                                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                        ┌────────────────────────────────────────────────┐
                        │       CALCULATION RESULT SUMMARY                │
                        ├────────────────────────────────────────────────┤
                        │                                                 │
                        │ Exposure ID: [Unique ID]                        │
                        │ Asset Class: [Classification]                   │
                        │ Calculation Approach: [FIRB/AIRB/SA]            │
                        │                                                 │
                        │ ── INPUT PARAMETERS ──                          │
                        │ EAD: €X,XXX,XXX                                 │
                        │ PD: Y.YY%                                       │
                        │ LGD: Z.ZZ%                                      │
                        │ Maturity: M years                               │
                        │ CRM Mitigation: [Collateral/Guarantee/None]     │
                        │                                                 │
                        │ ── CALCULATION PATH ──                          │
                        │ Rules Engine → Classified as [Type]             │
                        │ CRM Engine → Adjusted LGD: ZZ% (from Z%)        │
                        │ IRB Engine → RW: XX%, RWA: €XXX,XXX             │
                        │ Standardized Engine → RW: YY%, RWA: €YYY,YYY    │
                        │ Floor Validation → Floor: €FFF,FFF              │
                        │                                                 │
                        │ ── FINAL RESULT ──                              │
                        │ FINAL RWA: €FFF,FFF,FFF                         │
                        │ RISK WEIGHT: ZZ.Z%                              │
                        │ REGULATORY CAPITAL (8%): €CC,CCC,CCC            │
                        │ FLOOR BINDING: [YES/NO]                         │
                        │                                                 │
                        │ ── AUDIT TRAIL ──                               │
                        │ Calculation Date: [Date]                        │
                        │ Approach Selected: [FIRB/AIRB/SA]               │
                        │ Scalars Applied: [List]                         │
                        │ CRM Instruments: [List]                         │
                        │ Assumption Changes: [Notes]                     │
                        │                                                 │
                        └────────────────────────────────────────────────┘
```

---

## DETAILED SEQUENCING SUMMARY

### **Data Flow Sequence (Order of Execution)**

```
PARALLEL PATHS EXECUTED SIMULTANEOUSLY:

┌─ PATH 1: CLASSIFICATION & PREPARATION ──────────────────┐
│  1. Regulatory Mapping Engine                            │
│     Input: Borrower type, loan purpose, sector          │
│     Output: Asset class, available approaches            │
│                                                           │
│  2. Currency & Valuation Engine                          │
│     Input: Collateral details, property values          │
│     Output: Normalized EUR values, LTV, mismatch flags   │
│                                                           │
│  3. Rules Engine                                         │
│     Input: Asset class, borrower size, entity type      │
│     Output: Approach selected, scalar multipliers        │
│                                                           │
│  4. CRM Engine                                           │
│     Input: Collateral values, guarantees, haircuts      │
│     Output: Adjusted PD, LGD, EAD                        │
└────────────────────────────────────────────────────────┘

┌─ PATH 2: PARAMETER ENGINES (Parallel) ──────────────────┐
│  5a. EAD Engine                                          │
│      Input: Outstanding amount, commitments, CCF        │
│      Output: Final EAD                                   │
│                                                           │
│  5b. LGD Engine                                          │
│      Input: Asset class, collateral status, seniority   │
│      Output: Final LGD (with floors applied)             │
└────────────────────────────────────────────────────────┘

┌─ PATH 3: CALCULATION ENGINES (Parallel) ────────────────┐
│  6a. IRB Calculation Engine                              │
│      Input: PD, LGD, EAD, maturity, correlation         │
│      Output: RW, K value, RWA (if FIRB/AIRB)            │
│                                                           │
│  6b. Standardized Engine                                 │
│      Input: Asset class, rating, LTV (if property)      │
│      Output: Risk weight, RWA (always calculated)        │
└────────────────────────────────────────────────────────┘

┌─ PATH 4: FINAL VALIDATION ──────────────────────────────┐
│  7. Output Floor Engine                                  │
│     Input: IRB RWA, SA RWA, EL, provisions, flags      │
│     Output: FINAL RWA (max of IRB RWA & 72.5% floor)    │
└────────────────────────────────────────────────────────┘
```

### **Key Input Bypasses**

Some inputs skip ahead and bypass certain engines:

| Input | Engine | Bypasses | Reason |
|-------|--------|----------|--------|
| Outstanding Amount | EAD Engine | CRM initially | Used as-is, adjusted later by CRM |
| Maturity | IRB Engine | LGD/EAD Engines | Only needed for correlation & adjustment factor |
| Currency | Valuation Engine | Rules Engine | Normalized before routing |
| External Rating | Standardized Engine | IRB Engines | Direct RW lookup in SA |
| Property Value | Valuation Engine | CRM, LGD Engines | Normalized and used for LTV |
| Provisions | Output Floor Engine | All others | Only needed for final floor adjustment |

### **Critical Junction Points**

Where outputs become inputs:

1. **CRM Output → IRB/LGD Engines**: Adjusted PD, LGD, EAD flow downstream
2. **EAD/LGD Output → Both Calc Engines**: Both IRB and SA need these for RWA calculation
3. **Rules Output → All Calc Engines**: Approach selection + scalars determine which path and what multipliers
4. **Both Calc Engine Outputs → Floor Engine**: IRB RWA and SA RWA compared for floor application

