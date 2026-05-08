<script setup lang="ts">
type IncomeKey
  = | 'wageIncome'
    | 'taxableBenefits'
    | 'interestIncome'
    | 'longTermCapitalGains'
    | 'shortTermCapitalGains'
    | 'otherInvestmentIncome'

type AdjustmentKey
  = | 'retirement401kContributions'
    | 'healthInsurancePayments'
    | 'hsaContributions'

type OptionalDeductionKey
  = | 'mortgageInterest'
    | 'propertyTaxes'
    | 'stateTaxesWithholding'
    | 'investmentExpenses'
    | 'massachusettsRent'
    | 'carryOverCapitalLoss'

interface IncomeInput {
  key: IncomeKey
  label: string
  description: string
  examples?: string[]
  icon: string
}

interface AdjustmentInput {
  key: AdjustmentKey
  label: string
  description: string
  icon: string
}

interface OptionalDeductionInput {
  key: OptionalDeductionKey
  label: string
  description: string
  icon: string
}

interface TaxBracket {
  from: number
  to?: number
  rate: number
}

interface ConfigTaxBracket {
  from: number
  to?: number
  ratePercent: number
}

interface CalculatorConfig {
  federalStandardDeduction: number
  massachusettsSurtaxFloor: number
  socialSecurityMaximumTaxableWage: number
  federalTaxBrackets: ConfigTaxBracket[]
}

interface MarginalIncomeDeltas {
  wageIncome?: number
  otherInvestmentIncome?: number
  longTermCapitalGains?: number
}

type NumberInputValue = string | number

const incomeInputs: IncomeInput[] = [
  {
    key: 'wageIncome',
    label: 'Wage Income',
    description: 'W-2 wages, salary, bonuses, and earned compensation.',
    icon: 'i-lucide-briefcase-business'
  },
  {
    key: 'taxableBenefits',
    label: 'Taxable Benefits',
    description: 'Employer-provided benefits or fringe benefits treated as taxable income.',
    examples: ['Gym', 'Bike Share Memberships'],
    icon: 'i-lucide-badge-dollar-sign'
  },
  {
    key: 'interestIncome',
    label: 'Interest Income',
    description: 'Bank interest, CDs, Treasury interest, and taxable bond interest.',
    icon: 'i-lucide-landmark'
  },
  {
    key: 'longTermCapitalGains',
    label: 'Long Term Capital Gains and Qualified Dividends',
    description: 'Long-term capital gains plus dividends eligible for preferred tax rates.',
    icon: 'i-lucide-chart-line'
  },
  {
    key: 'shortTermCapitalGains',
    label: 'Short Term Capital Gains and Non-Qualified Dividends',
    description: 'Short-term capital gains plus dividends taxed at ordinary income rates.',
    icon: 'i-lucide-trending-up'
  },
  {
    key: 'otherInvestmentIncome',
    label: 'Other Investment Income',
    description: 'Rent, passive income, royalties, or other investment returns.',
    icon: 'i-lucide-wallet-cards'
  }
]

const adjustmentInputs: AdjustmentInput[] = [
  {
    key: 'retirement401kContributions',
    label: '401(k) Contributions',
    description: 'Pre-tax retirement contributions withheld from wages.',
    icon: 'i-lucide-piggy-bank'
  },
  {
    key: 'healthInsurancePayments',
    label: 'Health Insurance Payments',
    description: 'Premiums or payroll deductions for health coverage.',
    icon: 'i-lucide-heart-pulse'
  },
  {
    key: 'hsaContributions',
    label: 'HSA Contributions',
    description: 'Health savings account contributions for the tax year.',
    icon: 'i-lucide-shield-plus'
  }
]

const optionalDeductionInputs: OptionalDeductionInput[] = [
  {
    key: 'mortgageInterest',
    label: 'Mortgage Interest',
    description: 'Mortgage interest that may qualify for deduction treatment.',
    icon: 'i-lucide-house'
  },
  {
    key: 'propertyTaxes',
    label: 'Property Taxes',
    description: 'Property taxes paid that may qualify for deduction treatment.',
    icon: 'i-lucide-landmark'
  },
  {
    key: 'stateTaxesWithholding',
    label: 'State Taxes Withholding',
    description: 'State income taxes withheld during the tax year.',
    icon: 'i-lucide-file-minus'
  },
  {
    key: 'investmentExpenses',
    label: 'Investment Expenses',
    description: 'Eligible investment-related expenses for the tax year.',
    icon: 'i-lucide-receipt-text'
  },
  {
    key: 'massachusettsRent',
    label: 'Rent (Massachusetts)',
    description: 'Rent paid for Massachusetts deduction calculations.',
    icon: 'i-lucide-building-2'
  },
  {
    key: 'carryOverCapitalLoss',
    label: 'Carry-Over Capital Loss',
    description: 'Capital losses carried over from prior tax years.',
    icon: 'i-lucide-trending-down'
  }
]

const massachusettsPersonalExemption = 4400
const longTermTaxZeroRateThreshold = 49450
const longTermTaxFifteenRateThreshold = 545500
const defaultCalculatorConfig: CalculatorConfig = {
  federalStandardDeduction: 16100,
  massachusettsSurtaxFloor: 1107750,
  socialSecurityMaximumTaxableWage: 184500,
  federalTaxBrackets: [
    { from: 0, to: 12400, ratePercent: 10 },
    { from: 12400, to: 50400, ratePercent: 12 },
    { from: 50400, to: 105700, ratePercent: 22 },
    { from: 105700, to: 201775, ratePercent: 24 },
    { from: 201775, to: 256225, ratePercent: 32 },
    { from: 256225, to: 640600, ratePercent: 35 },
    { from: 640600, ratePercent: 37 }
  ]
}

const income = reactive<Record<IncomeKey, number | undefined>>({
  wageIncome: undefined,
  taxableBenefits: undefined,
  interestIncome: undefined,
  longTermCapitalGains: undefined,
  shortTermCapitalGains: undefined,
  otherInvestmentIncome: undefined
})

const adjustments = reactive<Record<AdjustmentKey, number | undefined>>({
  retirement401kContributions: undefined,
  healthInsurancePayments: undefined,
  hsaContributions: undefined
})

const optionalDeductions = reactive<Record<OptionalDeductionKey, number>>({
  mortgageInterest: 0,
  propertyTaxes: 0,
  stateTaxesWithholding: 0,
  investmentExpenses: 0,
  massachusettsRent: 0,
  carryOverCapitalLoss: 0
})

const calculatorConfig = reactive<CalculatorConfig>({
  federalStandardDeduction: defaultCalculatorConfig.federalStandardDeduction,
  massachusettsSurtaxFloor: defaultCalculatorConfig.massachusettsSurtaxFloor,
  socialSecurityMaximumTaxableWage: defaultCalculatorConfig.socialSecurityMaximumTaxableWage,
  federalTaxBrackets: defaultCalculatorConfig.federalTaxBrackets.map(bracket => ({ ...bracket }))
})

const interestBreakdownExpanded = ref(false)
const hsaBreakdownExpanded = ref(false)
const optionalDeductionsExpanded = ref(false)
const calculatorAssumptionsExpanded = ref(false)
const interestStateTaxExempt = ref<number | undefined>(undefined)
const hsaEmployerContributions = ref<number | undefined>(undefined)
const importInput = ref<HTMLInputElement | null>(null)

const normalizeAmount = (value: number | undefined) => Math.max(value ?? 0, 0)
const clampAmount = (value: number | undefined, max: number) => Math.min(normalizeAmount(value), max)

const interestIncomeTotal = computed(() => normalizeAmount(income.interestIncome))
const hsaContributionTotal = computed(() => normalizeAmount(adjustments.hsaContributions))

const stateTaxExemptInterest = computed({
  get: () => clampAmount(interestStateTaxExempt.value, interestIncomeTotal.value),
  set: (value: number | undefined) => {
    interestStateTaxExempt.value = clampAmount(value, interestIncomeTotal.value)
  }
})

const notStateTaxExemptInterest = computed({
  get: () => interestIncomeTotal.value - stateTaxExemptInterest.value,
  set: (value: number | undefined) => {
    interestStateTaxExempt.value = interestIncomeTotal.value - clampAmount(value, interestIncomeTotal.value)
  }
})

const hsaEmployerContributionAmount = computed({
  get: () => clampAmount(hsaEmployerContributions.value, hsaContributionTotal.value),
  set: (value: number | undefined) => {
    hsaEmployerContributions.value = clampAmount(value, hsaContributionTotal.value)
  }
})

const hsaEmployeeContributionAmount = computed({
  get: () => hsaContributionTotal.value - hsaEmployerContributionAmount.value,
  set: (value: number | undefined) => {
    hsaEmployerContributions.value = hsaContributionTotal.value - clampAmount(value, hsaContributionTotal.value)
  }
})

watch(interestIncomeTotal, (total) => {
  interestStateTaxExempt.value = clampAmount(interestStateTaxExempt.value, total)
})

watch(hsaContributionTotal, (total) => {
  hsaEmployerContributions.value = clampAmount(hsaEmployerContributions.value, total)
})

const formatCurrency = (value: number) =>
  new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0
  }).format(value)

const formatNumberInput = (value: number | undefined) =>
  new Intl.NumberFormat('en-US', {
    maximumFractionDigits: 2
  }).format(value ?? 0)

const parseNumberInput = (value: NumberInputValue) => {
  const parsedValue = Number(String(value).replaceAll(',', ''))

  return Number.isFinite(parsedValue) ? parsedValue : 0
}

const parseOptionalNumberInput = (value: NumberInputValue) => {
  const stringValue = String(value).trim()

  if (!stringValue) {
    return undefined
  }

  return parseNumberInput(stringValue)
}

const formatPercent = (value: number) =>
  new Intl.NumberFormat('en-US', {
    style: 'percent',
    minimumFractionDigits: 2,
    maximumFractionDigits: 2
  }).format(value)

const federalOrdinaryTaxBrackets = computed<TaxBracket[]>(() =>
  calculatorConfig.federalTaxBrackets.map(bracket => ({
    from: normalizeAmount(bracket.from),
    to: bracket.to === undefined ? undefined : normalizeAmount(bracket.to),
    rate: normalizeAmount(bracket.ratePercent) / 100
  }))
)

const resetCalculatorConfig = () => {
  calculatorConfig.federalStandardDeduction = defaultCalculatorConfig.federalStandardDeduction
  calculatorConfig.massachusettsSurtaxFloor = defaultCalculatorConfig.massachusettsSurtaxFloor
  calculatorConfig.socialSecurityMaximumTaxableWage = defaultCalculatorConfig.socialSecurityMaximumTaxableWage
  calculatorConfig.federalTaxBrackets.splice(
    0,
    calculatorConfig.federalTaxBrackets.length,
    ...defaultCalculatorConfig.federalTaxBrackets.map(bracket => ({ ...bracket }))
  )
}

const addFederalTaxBracket = () => {
  const lastBracket = calculatorConfig.federalTaxBrackets.at(-1)

  calculatorConfig.federalTaxBrackets.push({
    from: lastBracket?.to ?? lastBracket?.from ?? 0,
    ratePercent: lastBracket?.ratePercent ?? 0
  })
}

const removeFederalTaxBracket = (index: number) => {
  if (calculatorConfig.federalTaxBrackets.length <= 1) {
    return
  }

  calculatorConfig.federalTaxBrackets.splice(index, 1)
}

const calculateProgressiveTax = (taxableIncome: number, brackets: TaxBracket[]) => {
  const incomeAmount = Math.max(taxableIncome, 0)

  return brackets.reduce((tax, bracket) => {
    const bracketTop = bracket.to ?? incomeAmount
    const taxableInBracket = Math.max(Math.min(incomeAmount, bracketTop) - bracket.from, 0)

    return tax + taxableInBracket * bracket.rate
  }, 0)
}

const optionalDeductionTotalPlaceholder = computed(() => {
  const mortgageInterest = optionalDeductions.mortgageInterest
  const propertyTaxes = optionalDeductions.propertyTaxes
  const stateTaxesWithholding = optionalDeductions.stateTaxesWithholding
  const investmentExpenses = optionalDeductions.investmentExpenses
  const massachusettsRent = optionalDeductions.massachusettsRent
  const carryOverCapitalLoss = optionalDeductions.carryOverCapitalLoss

  // TODO: Replace with your actual optional deduction formula.
  return mortgageInterest + propertyTaxes + stateTaxesWithholding + investmentExpenses + massachusettsRent + carryOverCapitalLoss
})

const grossIncome = computed(() => {
  const wageIncome = income.wageIncome ?? 0
  const taxableBenefits = income.taxableBenefits ?? 0
  const interestIncome = income.interestIncome ?? 0
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0
  const otherInvestmentIncome = income.otherInvestmentIncome ?? 0

  return wageIncome + taxableBenefits + interestIncome + shortTermCapitalGains + otherInvestmentIncome
})

const federalOrdinaryIncome = computed(() => {
  const retirement401kContributions = adjustments.retirement401kContributions ?? 0
  const healthInsurancePayments = adjustments.healthInsurancePayments ?? 0
  const investmentExpenses = optionalDeductions.investmentExpenses
  const stateTaxesWithholding = optionalDeductions.stateTaxesWithholding
  const propertyTaxes = optionalDeductions.propertyTaxes
  const mortgageInterest = optionalDeductions.mortgageInterest
  const carryOverCapitalLoss = optionalDeductions.carryOverCapitalLoss
  const longTermCapitalGains = income.longTermCapitalGains ?? 0
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0
  const stateAndPropertyTaxDeduction = Math.min(stateTaxesWithholding + propertyTaxes, 40000)
  const itemizedOrStandardDeduction = Math.max(
    calculatorConfig.federalStandardDeduction,
    stateAndPropertyTaxDeduction + mortgageInterest
  )
  const capitalLossDeduction = Math.min(
    3000,
    Math.max(0, carryOverCapitalLoss - (longTermCapitalGains + shortTermCapitalGains))
  )

  const taxableIncome = grossIncome.value
    - retirement401kContributions
    - hsaEmployeeContributionAmount.value
    - investmentExpenses
    - healthInsurancePayments
    - itemizedOrStandardDeduction
    - capitalLossDeduction

  return Math.max(taxableIncome, 0)
})

const longTermTaxableIncome = computed(() => income.longTermCapitalGains ?? 0)

const netInvestmentTaxableIncome = computed(() => {
  const interestIncome = income.interestIncome ?? 0
  const otherInvestmentIncome = income.otherInvestmentIncome ?? 0
  const longTermCapitalGains = income.longTermCapitalGains ?? 0
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0

  return interestIncome + otherInvestmentIncome + longTermCapitalGains + shortTermCapitalGains
})

const ficaWage = computed(() => {
  const wageIncome = income.wageIncome ?? 0
  const taxableBenefits = income.taxableBenefits ?? 0
  const healthInsurancePayments = adjustments.healthInsurancePayments ?? 0

  return Math.max(
    wageIncome + taxableBenefits - hsaEmployeeContributionAmount.value - healthInsurancePayments,
    0
  )
})

const ficaTax = computed(() => {
  const ficaTaxableIncome = ficaWage.value

  return Math.min(calculatorConfig.socialSecurityMaximumTaxableWage * 0.062, 0.062 * ficaTaxableIncome)
    + 0.0145 * ficaTaxableIncome
    + 0.009 * Math.max(ficaTaxableIncome - 200000, 0)
})

const massachusettsTaxableIncome = computed(() => {
  const retirement401kContributions = adjustments.retirement401kContributions ?? 0
  const healthInsurancePayments = adjustments.healthInsurancePayments ?? 0
  const investmentExpenses = optionalDeductions.investmentExpenses
  const massachusettsRent = optionalDeductions.massachusettsRent
  const carryOverCapitalLoss = optionalDeductions.carryOverCapitalLoss
  const longTermCapitalGains = income.longTermCapitalGains ?? 0
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0
  const capitalLossAfterGains = Math.max(
    carryOverCapitalLoss - (longTermCapitalGains + shortTermCapitalGains),
    0
  )
  const interestCapitalLossDeduction = Math.min(
    notStateTaxExemptInterest.value,
    capitalLossAfterGains,
    2000
  )

  const taxableIncome = grossIncome.value
    + longTermTaxableIncome.value
    - retirement401kContributions
    - hsaEmployeeContributionAmount.value
    - healthInsurancePayments
    - investmentExpenses
    - Math.min(massachusettsRent, 3000)
    - Math.min(ficaTax.value, 2000)
    - massachusettsPersonalExemption
    - interestCapitalLossDeduction
    - stateTaxExemptInterest.value

  return Math.max(taxableIncome, 0)
})

const federalTax = computed(() =>
  calculateProgressiveTax(federalOrdinaryIncome.value, federalOrdinaryTaxBrackets.value)
)

const longTermTax = computed(() => {
  const federalIncome = federalOrdinaryIncome.value
  const longTermIncome = longTermTaxableIncome.value

  if (federalIncome <= longTermTaxZeroRateThreshold) {
    return longTermIncome * 0
  }

  if (federalIncome <= longTermTaxFifteenRateThreshold) {
    return longTermIncome * 0.15
  }

  return longTermIncome * 0.20
})

const stateTax = computed(() => {
  const taxableIncome = Math.max(massachusettsTaxableIncome.value, 0)

  return Math.min(taxableIncome, calculatorConfig.massachusettsSurtaxFloor) * 0.05
    + Math.max(taxableIncome - calculatorConfig.massachusettsSurtaxFloor, 0) * 0.09
})

const netInvestmentIncomeTax = computed(() => netInvestmentTaxableIncome.value * 0.038)

const totalTax = computed(() =>
  federalTax.value + stateTax.value + ficaTax.value + netInvestmentIncomeTax.value + longTermTax.value
)

const totalIncome = computed(() => {
  const wageIncome = income.wageIncome ?? 0
  const taxableBenefits = income.taxableBenefits ?? 0
  const interestIncome = income.interestIncome ?? 0
  const longTermCapitalGains = income.longTermCapitalGains ?? 0
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0
  const otherInvestmentIncome = income.otherInvestmentIncome ?? 0

  return wageIncome
    + taxableBenefits
    + interestIncome
    + longTermCapitalGains
    + shortTermCapitalGains
    + otherInvestmentIncome
})

const netIncome = computed(() => {
  const retirement401kContributions = adjustments.retirement401kContributions ?? 0
  const healthInsurancePayments = adjustments.healthInsurancePayments ?? 0
  const taxableBenefits = income.taxableBenefits ?? 0

  return totalIncome.value
    - taxableBenefits
    - retirement401kContributions
    - hsaEmployeeContributionAmount.value
    - healthInsurancePayments
    - totalTax.value
})

const netIncomeMinusInvestmentIncome = computed(() => {
  const interestIncome = income.interestIncome ?? 0
  const longTermCapitalGains = income.longTermCapitalGains ?? 0
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0
  const otherInvestmentIncome = income.otherInvestmentIncome ?? 0

  return netIncome.value
    - shortTermCapitalGains
    - longTermCapitalGains
    - otherInvestmentIncome
    - interestIncome
})

const effectiveTaxRate = computed(() => {
  if (totalIncome.value <= 0) {
    return 0
  }

  return totalTax.value / totalIncome.value
})

const calculateNetIncomeWithDeltas = (deltas: MarginalIncomeDeltas) => {
  const wageIncome = (income.wageIncome ?? 0) + (deltas.wageIncome ?? 0)
  const taxableBenefits = income.taxableBenefits ?? 0
  const interestIncome = income.interestIncome ?? 0
  const longTermCapitalGains = (income.longTermCapitalGains ?? 0) + (deltas.longTermCapitalGains ?? 0)
  const shortTermCapitalGains = income.shortTermCapitalGains ?? 0
  const otherInvestmentIncome = (income.otherInvestmentIncome ?? 0) + (deltas.otherInvestmentIncome ?? 0)
  const retirement401kContributions = adjustments.retirement401kContributions ?? 0
  const healthInsurancePayments = adjustments.healthInsurancePayments ?? 0
  const investmentExpenses = optionalDeductions.investmentExpenses
  const stateTaxesWithholding = optionalDeductions.stateTaxesWithholding
  const propertyTaxes = optionalDeductions.propertyTaxes
  const mortgageInterest = optionalDeductions.mortgageInterest
  const massachusettsRent = optionalDeductions.massachusettsRent
  const carryOverCapitalLoss = optionalDeductions.carryOverCapitalLoss
  const hsaEmployeeContributions = hsaEmployeeContributionAmount.value
  const grossIncomeAmount = wageIncome
    + taxableBenefits
    + interestIncome
    + shortTermCapitalGains
    + otherInvestmentIncome
  const totalIncomeAmount = grossIncomeAmount + longTermCapitalGains
  const stateAndPropertyTaxDeduction = Math.min(stateTaxesWithholding + propertyTaxes, 40000)
  const itemizedOrStandardDeduction = Math.max(
    calculatorConfig.federalStandardDeduction,
    stateAndPropertyTaxDeduction + mortgageInterest
  )
  const capitalLossAfterGains = Math.max(
    carryOverCapitalLoss - (longTermCapitalGains + shortTermCapitalGains),
    0
  )
  const federalCapitalLossDeduction = Math.min(3000, capitalLossAfterGains)
  const federalOrdinaryIncomeAmount = Math.max(grossIncomeAmount
    - retirement401kContributions
    - hsaEmployeeContributions
    - investmentExpenses
    - healthInsurancePayments
    - itemizedOrStandardDeduction
    - federalCapitalLossDeduction,
  0)
  const ficaTaxableIncome = Math.max(wageIncome
    + taxableBenefits
    - hsaEmployeeContributions
    - healthInsurancePayments,
  0)
  const ficaTaxAmount = Math.min(
    calculatorConfig.socialSecurityMaximumTaxableWage * 0.062,
    0.062 * ficaTaxableIncome
  ) + (0.0145 * ficaTaxableIncome) + (0.009 * Math.max(ficaTaxableIncome - 200000, 0))
  const federalTaxAmount = calculateProgressiveTax(
    federalOrdinaryIncomeAmount,
    federalOrdinaryTaxBrackets.value
  )
  const longTermTaxRate = federalOrdinaryIncomeAmount <= longTermTaxZeroRateThreshold
    ? 0
    : federalOrdinaryIncomeAmount <= longTermTaxFifteenRateThreshold
      ? 0.15
      : 0.20
  const longTermTaxAmount = longTermCapitalGains * longTermTaxRate
  const interestCapitalLossDeduction = Math.min(
    notStateTaxExemptInterest.value,
    capitalLossAfterGains,
    2000
  )
  const massachusettsTaxableIncomeAmount = Math.max(grossIncomeAmount
    + longTermCapitalGains
    - retirement401kContributions
    - hsaEmployeeContributions
    - healthInsurancePayments
    - investmentExpenses
    - Math.min(massachusettsRent, 3000)
    - Math.min(ficaTaxAmount, 2000)
    - massachusettsPersonalExemption
    - interestCapitalLossDeduction
    - stateTaxExemptInterest.value,
  0)
  const stateTaxAmount = Math.min(
    Math.max(massachusettsTaxableIncomeAmount, 0),
    calculatorConfig.massachusettsSurtaxFloor
  ) * 0.05 + Math.max(massachusettsTaxableIncomeAmount - calculatorConfig.massachusettsSurtaxFloor, 0) * 0.09
  const netInvestmentTaxableIncomeAmount = interestIncome
    + otherInvestmentIncome
    + longTermCapitalGains
    + shortTermCapitalGains
  const netInvestmentIncomeTaxAmount = netInvestmentTaxableIncomeAmount * 0.038
  const totalTaxAmount = federalTaxAmount
    + stateTaxAmount
    + ficaTaxAmount
    + netInvestmentIncomeTaxAmount
    + longTermTaxAmount

  return totalIncomeAmount
    - taxableBenefits
    - retirement401kContributions
    - hsaEmployeeContributions
    - healthInsurancePayments
    - totalTaxAmount
}

const calculateMarginalTaxRate = (deltas: MarginalIncomeDeltas) => {
  const incomeDelta = Object.values(deltas).reduce((total, delta) => total + (delta ?? 0), 0)

  if (incomeDelta <= 0) {
    return 0
  }

  const netIncomeDelta = calculateNetIncomeWithDeltas(deltas) - calculateNetIncomeWithDeltas({})

  return 1 - netIncomeDelta / incomeDelta
}

const marginalTaxRateWageIncome = computed(() => calculateMarginalTaxRate({ wageIncome: 1 }))
const marginalTaxRateInvestmentIncome = computed(() => calculateMarginalTaxRate({ otherInvestmentIncome: 1 }))
const marginalTaxRateLongTermIncome = computed(() => calculateMarginalTaxRate({ longTermCapitalGains: 1 }))

const exportInputs = () => {
  const data = {
    version: 1,
    exportedAt: new Date().toISOString(),
    income: { ...income },
    adjustments: { ...adjustments },
    optionalDeductions: { ...optionalDeductions },
    calculatorConfig: {
      federalStandardDeduction: calculatorConfig.federalStandardDeduction,
      massachusettsSurtaxFloor: calculatorConfig.massachusettsSurtaxFloor,
      socialSecurityMaximumTaxableWage: calculatorConfig.socialSecurityMaximumTaxableWage,
      federalTaxBrackets: calculatorConfig.federalTaxBrackets.map(bracket => ({ ...bracket }))
    },
    interestBreakdown: {
      stateTaxExempt: stateTaxExemptInterest.value,
      notStateTaxExempt: notStateTaxExemptInterest.value
    },
    hsaBreakdown: {
      employeeContributions: hsaEmployeeContributionAmount.value,
      employerContributions: hsaEmployerContributionAmount.value
    }
  }

  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = `tax-calculator-inputs-${new Date().toISOString().slice(0, 10)}.json`
  link.click()
  URL.revokeObjectURL(url)
}

const importInputs = async (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]

  if (!file) {
    return
  }

  try {
    const importedData = JSON.parse(await file.text()) as {
      income?: Partial<Record<IncomeKey, number>>
      adjustments?: Partial<Record<AdjustmentKey, number>>
      optionalDeductions?: Partial<Record<OptionalDeductionKey, number>>
      calculatorConfig?: Partial<CalculatorConfig>
      interestBreakdown?: {
        stateTaxExempt?: number
        notStateTaxExempt?: number
      }
      hsaBreakdown?: {
        employeeContributions?: number
        employerContributions?: number
      }
    }

    for (const input of incomeInputs) {
      income[input.key] = importedData.income?.[input.key]
    }

    for (const input of adjustmentInputs) {
      adjustments[input.key] = importedData.adjustments?.[input.key]
    }

    for (const input of optionalDeductionInputs) {
      optionalDeductions[input.key] = importedData.optionalDeductions?.[input.key] ?? 0
    }

    if (importedData.calculatorConfig) {
      calculatorConfig.federalStandardDeduction = importedData.calculatorConfig.federalStandardDeduction
        ?? defaultCalculatorConfig.federalStandardDeduction
      calculatorConfig.massachusettsSurtaxFloor = importedData.calculatorConfig.massachusettsSurtaxFloor
        ?? defaultCalculatorConfig.massachusettsSurtaxFloor
      calculatorConfig.socialSecurityMaximumTaxableWage = importedData.calculatorConfig.socialSecurityMaximumTaxableWage
        ?? defaultCalculatorConfig.socialSecurityMaximumTaxableWage
      calculatorConfig.federalTaxBrackets.splice(
        0,
        calculatorConfig.federalTaxBrackets.length,
        ...(importedData.calculatorConfig.federalTaxBrackets ?? defaultCalculatorConfig.federalTaxBrackets)
          .map(bracket => ({ ...bracket }))
      )
    }

    stateTaxExemptInterest.value = importedData.interestBreakdown?.stateTaxExempt ?? 0

    if (importedData.hsaBreakdown) {
      hsaEmployerContributionAmount.value = importedData.hsaBreakdown.employerContributions
        ?? hsaContributionTotal.value - normalizeAmount(importedData.hsaBreakdown.employeeContributions)
    } else {
      hsaEmployerContributionAmount.value = 0
    }

    interestBreakdownExpanded.value = !!importedData.interestBreakdown
    hsaBreakdownExpanded.value = !!importedData.hsaBreakdown
    optionalDeductionsExpanded.value = !!importedData.optionalDeductions
    calculatorAssumptionsExpanded.value = !!importedData.calculatorConfig
  } finally {
    target.value = ''
  }
}
</script>

<template>
  <UContainer class="py-8 sm:py-10">
    <div class="mb-8">
      <div class="max-w-4xl">
        <h1 class="text-3xl font-semibold tracking-normal text-highlighted sm:text-4xl">
          Income Tax Calculator (Massachusetts)
        </h1>

        <p class="mt-3 text-base leading-7 text-muted">
          To calculate your net income and state and federal taxes, enter income, contributions, and deductions amounts below. The tax calculator is built assuming a "Single" Filing Status for a full-year resident of the Commonwealth of Massachusetts.
        </p>
      </div>
    </div>

    <div class="grid gap-6 lg:grid-cols-[minmax(0,1fr)_360px]">
      <div class="space-y-6">
        <section class="rounded-lg border border-default bg-default">
          <div class="border-b border-default px-5 py-4 sm:px-6">
            <h2 class="text-lg font-semibold text-highlighted">
              Income Details
            </h2>
          </div>

          <div class="divide-y divide-default">
            <div
              v-for="input in incomeInputs"
              :key="input.key"
              class="grid gap-4 px-5 py-5 sm:grid-cols-[1fr_240px] sm:items-center sm:px-6"
            >
              <div class="flex gap-3">
                <div class="flex size-10 shrink-0 items-center justify-center rounded-md bg-primary/10 text-primary">
                  <UIcon
                    :name="input.icon"
                    class="size-5"
                  />
                </div>

                <div>
                  <UFormField
                    :label="input.label"
                    :name="input.key"
                    class="[&>label]:text-base [&>label]:font-semibold"
                  >
                    <p class="mt-1 text-sm leading-6 text-muted">
                      {{ input.description }}
                      <template v-if="input.examples?.length">
                        Examples:
                        <span
                          v-for="(example, index) in input.examples"
                          :key="example"
                        >
                          {{ example }}{{ index === input.examples.length - 1 ? '.' : ', ' }}
                        </span>
                      </template>
                    </p>
                  </UFormField>
                </div>
              </div>

              <UInput
                :model-value="formatNumberInput(income[input.key])"
                type="text"
                :min="0"
                :step="100"
                inputmode="decimal"
                placeholder="0"
                icon="i-lucide-dollar-sign"
                size="xl"
                class="w-full"
                @update:model-value="income[input.key] = parseNumberInput($event)"
              />

              <div
                v-if="input.key === 'interestIncome'"
                class="sm:col-span-2"
              >
                <UButton
                  :icon="interestBreakdownExpanded ? 'i-lucide-chevron-up' : 'i-lucide-chevron-down'"
                  color="neutral"
                  variant="ghost"
                  size="sm"
                  class="px-0"
                  :label="interestBreakdownExpanded ? 'Hide interest breakdown' : 'Show interest breakdown'"
                  @click="interestBreakdownExpanded = !interestBreakdownExpanded"
                />

                <div
                  v-if="interestBreakdownExpanded"
                  class="mt-3 grid gap-3 rounded-lg border border-default bg-muted/30 p-4 sm:grid-cols-2"
                >
                  <UFormField
                    label="State Tax Exempt"
                    name="stateTaxExemptInterest"
                  >
                    <UInput
                      :model-value="formatNumberInput(stateTaxExemptInterest)"
                      type="text"
                      :min="0"
                      :max="interestIncomeTotal"
                      :step="100"
                      inputmode="decimal"
                      placeholder="0"
                      icon="i-lucide-dollar-sign"
                      class="w-full"
                      @update:model-value="stateTaxExemptInterest = parseNumberInput($event)"
                    />
                  </UFormField>

                  <UFormField
                    label="Not State Tax Exempt"
                    name="notStateTaxExemptInterest"
                  >
                    <UInput
                      :model-value="formatNumberInput(notStateTaxExemptInterest)"
                      type="text"
                      :min="0"
                      :max="interestIncomeTotal"
                      :step="100"
                      inputmode="decimal"
                      placeholder="0"
                      icon="i-lucide-dollar-sign"
                      class="w-full"
                      @update:model-value="notStateTaxExemptInterest = parseNumberInput($event)"
                    />
                  </UFormField>

                  <p class="text-sm text-muted sm:col-span-2">
                    Breakdown total: {{ formatCurrency(stateTaxExemptInterest + notStateTaxExemptInterest) }}
                  </p>
                </div>
              </div>
            </div>
          </div>
        </section>

        <section class="rounded-lg border border-default bg-default">
          <div class="border-b border-default px-5 py-4 sm:px-6">
            <h2 class="text-lg font-semibold text-highlighted">
              Adjustments & Contributions
            </h2>
          </div>

          <div class="divide-y divide-default">
            <div
              v-for="input in adjustmentInputs"
              :key="input.key"
              class="grid gap-4 px-5 py-5 sm:grid-cols-[1fr_240px] sm:items-center sm:px-6"
            >
              <div class="flex gap-3">
                <div class="flex size-10 shrink-0 items-center justify-center rounded-md bg-info/10 text-info">
                  <UIcon
                    :name="input.icon"
                    class="size-5"
                  />
                </div>

                <div>
                  <UFormField
                    :label="input.label"
                    :name="input.key"
                    class="[&>label]:text-base [&>label]:font-semibold"
                  >
                    <p class="mt-1 text-sm leading-6 text-muted">
                      {{ input.description }}
                    </p>
                  </UFormField>
                </div>
              </div>

              <UInput
                :model-value="formatNumberInput(adjustments[input.key])"
                type="text"
                :min="0"
                :step="100"
                inputmode="decimal"
                placeholder="0"
                icon="i-lucide-dollar-sign"
                size="xl"
                class="w-full"
                @update:model-value="adjustments[input.key] = parseNumberInput($event)"
              />

              <div
                v-if="input.key === 'hsaContributions'"
                class="sm:col-span-2"
              >
                <UButton
                  :icon="hsaBreakdownExpanded ? 'i-lucide-chevron-up' : 'i-lucide-chevron-down'"
                  color="neutral"
                  variant="ghost"
                  size="sm"
                  class="px-0"
                  :label="hsaBreakdownExpanded ? 'Hide HSA breakdown' : 'Show HSA breakdown'"
                  @click="hsaBreakdownExpanded = !hsaBreakdownExpanded"
                />

                <div
                  v-if="hsaBreakdownExpanded"
                  class="mt-3 grid gap-3 rounded-lg border border-default bg-muted/30 p-4 sm:grid-cols-2"
                >
                  <UFormField
                    label="Employee Contributions"
                    name="hsaEmployeeContributions"
                  >
                    <UInput
                      :model-value="formatNumberInput(hsaEmployeeContributionAmount)"
                      type="text"
                      :min="0"
                      :max="hsaContributionTotal"
                      :step="100"
                      inputmode="decimal"
                      placeholder="0"
                      icon="i-lucide-dollar-sign"
                      class="w-full"
                      @update:model-value="hsaEmployeeContributionAmount = parseNumberInput($event)"
                    />
                  </UFormField>

                  <UFormField
                    label="Employer Contributions"
                    name="hsaEmployerContributions"
                  >
                    <UInput
                      :model-value="formatNumberInput(hsaEmployerContributionAmount)"
                      type="text"
                      :min="0"
                      :max="hsaContributionTotal"
                      :step="100"
                      inputmode="decimal"
                      placeholder="0"
                      icon="i-lucide-dollar-sign"
                      class="w-full"
                      @update:model-value="hsaEmployerContributionAmount = parseNumberInput($event)"
                    />
                  </UFormField>

                  <p class="text-sm text-muted sm:col-span-2">
                    Breakdown total: {{ formatCurrency(hsaEmployeeContributionAmount + hsaEmployerContributionAmount) }}
                  </p>
                </div>
              </div>
            </div>
          </div>
        </section>

        <section class="rounded-lg border border-default bg-default">
          <button
            type="button"
            class="flex w-full items-center justify-between gap-3 border-b border-default px-5 py-4 text-left sm:px-6"
            @click="optionalDeductionsExpanded = !optionalDeductionsExpanded"
          >
            <span>
              <span class="block text-lg font-semibold text-highlighted">
                Optional Deductions
              </span>
              <span class="mt-1 block text-sm text-muted">
                {{ formatCurrency(optionalDeductionTotalPlaceholder) }} currently entered
              </span>
            </span>

            <UIcon
              :name="optionalDeductionsExpanded ? 'i-lucide-chevron-up' : 'i-lucide-chevron-down'"
              class="size-5 shrink-0 text-muted"
            />
          </button>

          <div
            v-if="optionalDeductionsExpanded"
            class="divide-y divide-default"
          >
            <div
              v-for="input in optionalDeductionInputs"
              :key="input.key"
              class="grid gap-4 px-5 py-5 sm:grid-cols-[1fr_240px] sm:items-center sm:px-6"
            >
              <div class="flex gap-3">
                <div class="flex size-10 shrink-0 items-center justify-center rounded-md bg-warning/10 text-warning">
                  <UIcon
                    :name="input.icon"
                    class="size-5"
                  />
                </div>

                <div>
                  <UFormField
                    :label="input.label"
                    :name="input.key"
                    class="[&>label]:text-base [&>label]:font-semibold"
                  >
                    <p class="mt-1 text-sm leading-6 text-muted">
                      {{ input.description }}
                    </p>
                  </UFormField>
                </div>
              </div>

              <UInput
                :model-value="formatNumberInput(optionalDeductions[input.key])"
                type="text"
                :min="0"
                :step="100"
                inputmode="decimal"
                placeholder="0"
                icon="i-lucide-dollar-sign"
                size="xl"
                class="w-full"
                @update:model-value="optionalDeductions[input.key] = parseNumberInput($event)"
              />
            </div>
          </div>
        </section>

        <section class="rounded-lg border border-default bg-default">
          <button
            type="button"
            class="flex w-full items-center justify-between gap-3 border-b border-default px-5 py-4 text-left sm:px-6"
            @click="calculatorAssumptionsExpanded = !calculatorAssumptionsExpanded"
          >
            <span>
              <span class="block text-lg font-semibold text-highlighted">
                Advanced Configurations
              </span>
              <span class="mt-1 block text-sm text-muted">
                Federal brackets, standard deduction, Massachusetts surtax floor, and Social Security wage base
              </span>
            </span>

            <UIcon
              :name="calculatorAssumptionsExpanded ? 'i-lucide-chevron-up' : 'i-lucide-chevron-down'"
              class="size-5 shrink-0 text-muted"
            />
          </button>

          <div
            v-if="calculatorAssumptionsExpanded"
            class="space-y-6 px-5 py-5 sm:px-6"
          >
            <div class="grid gap-4 md:grid-cols-3">
              <UFormField
                label="Federal Standard Deduction"
                name="federalStandardDeduction"
                class="[&>label]:flex [&>label]:min-h-10 [&>label]:items-end"
              >
                <UInput
                  :model-value="formatNumberInput(calculatorConfig.federalStandardDeduction)"
                  type="text"
                  :min="0"
                  :step="100"
                  inputmode="decimal"
                  icon="i-lucide-dollar-sign"
                  class="w-full"
                  @update:model-value="calculatorConfig.federalStandardDeduction = parseNumberInput($event)"
                />
              </UFormField>

              <UFormField
                label="Massachusetts Surtax Floor"
                name="massachusettsSurtaxFloor"
                class="[&>label]:flex [&>label]:min-h-10 [&>label]:items-end"
              >
                <UInput
                  :model-value="formatNumberInput(calculatorConfig.massachusettsSurtaxFloor)"
                  type="text"
                  :min="0"
                  :step="1000"
                  inputmode="decimal"
                  icon="i-lucide-dollar-sign"
                  class="w-full"
                  @update:model-value="calculatorConfig.massachusettsSurtaxFloor = parseNumberInput($event)"
                />
              </UFormField>

              <UFormField
                label="Social Security Wage Ceiling"
                name="socialSecurityMaximumTaxableWage"
                class="[&>label]:flex [&>label]:min-h-10 [&>label]:items-end"
              >
                <UInput
                  :model-value="formatNumberInput(calculatorConfig.socialSecurityMaximumTaxableWage)"
                  type="text"
                  :min="0"
                  :step="100"
                  inputmode="decimal"
                  icon="i-lucide-dollar-sign"
                  class="w-full"
                  @update:model-value="calculatorConfig.socialSecurityMaximumTaxableWage = parseNumberInput($event)"
                />
              </UFormField>
            </div>

            <div>
              <div class="mb-3 flex items-center justify-between gap-3">
                <h3 class="text-sm font-semibold text-highlighted">
                  Federal Tax Brackets
                </h3>

                <div class="flex items-center gap-2">
                  <UButton
                    label="Reset"
                    icon="i-lucide-rotate-ccw"
                    color="neutral"
                    variant="ghost"
                    size="sm"
                    @click="resetCalculatorConfig"
                  />

                  <UButton
                    label="Add Bracket"
                    icon="i-lucide-plus"
                    color="neutral"
                    variant="subtle"
                    size="sm"
                    @click="addFederalTaxBracket"
                  />
                </div>
              </div>

              <div class="overflow-x-auto">
                <table class="w-full min-w-[640px] text-left text-sm">
                  <thead class="border-b border-default text-muted">
                    <tr>
                      <th class="py-2 pr-3 font-medium">
                        From
                      </th>
                      <th class="px-3 py-2 font-medium">
                        To
                      </th>
                      <th class="px-3 py-2 font-medium">
                        Rate %
                      </th>
                      <th class="py-2 pl-3 text-right font-medium">
                        Actions
                      </th>
                    </tr>
                  </thead>

                  <tbody class="divide-y divide-default">
                    <tr
                      v-for="(bracket, index) in calculatorConfig.federalTaxBrackets"
                      :key="index"
                    >
                      <td class="py-3 pr-3">
                        <UInput
                          :model-value="formatNumberInput(bracket.from)"
                          type="text"
                          :min="0"
                          :step="100"
                          inputmode="decimal"
                          icon="i-lucide-dollar-sign"
                          class="w-full"
                          @update:model-value="bracket.from = parseNumberInput($event)"
                        />
                      </td>

                      <td class="px-3 py-3">
                        <UInput
                          :model-value="bracket.to === undefined ? '' : formatNumberInput(bracket.to)"
                          type="text"
                          :min="0"
                          :step="100"
                          inputmode="decimal"
                          placeholder="No cap"
                          icon="i-lucide-dollar-sign"
                          class="w-full"
                          @update:model-value="bracket.to = parseOptionalNumberInput($event)"
                        />
                      </td>

                      <td class="px-3 py-3">
                        <UInput
                          v-model.number="bracket.ratePercent"
                          type="number"
                          :min="0"
                          :step="0.01"
                          inputmode="decimal"
                          class="w-full"
                        />
                      </td>

                      <td class="py-3 pl-3 text-right">
                        <UButton
                          icon="i-lucide-trash-2"
                          color="neutral"
                          variant="ghost"
                          size="sm"
                          aria-label="Remove federal tax bracket"
                          :disabled="calculatorConfig.federalTaxBrackets.length <= 1"
                          @click="removeFederalTaxBracket(index)"
                        />
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        </section>
      </div>

      <aside class="space-y-4">
        <section class="rounded-lg border border-default bg-default p-5">
          <div class="flex items-center justify-between gap-3">
            <h2 class="text-lg font-semibold text-highlighted">
              Load & Save Calculator Inputs
            </h2>

            <UIcon
              name="i-lucide-file-json"
              class="size-5 text-muted"
            />
          </div>

          <div class="mt-5 grid gap-3">
            <UButton
              label="Import JSON"
              icon="i-lucide-upload"
              color="neutral"
              variant="subtle"
              block
              @click="importInput?.click()"
            />

            <UButton
              label="Export JSON"
              icon="i-lucide-download"
              color="primary"
              variant="solid"
              block
              @click="exportInputs"
            />
          </div>

          <input
            ref="importInput"
            type="file"
            accept="application/json,.json"
            class="hidden"
            @change="importInputs"
          >
        </section>

        <section class="rounded-lg border border-default bg-default p-5">
          <div class="flex items-center justify-between gap-3">
            <h2 class="text-lg font-semibold text-highlighted">
              Calculations
            </h2>

            <UIcon
              name="i-lucide-braces"
              class="size-5 text-muted"
            />
          </div>

          <div class="mt-5 space-y-6">
            <div>
              <h3 class="text-sm font-semibold text-highlighted">
                Taxable Income
              </h3>

              <dl class="mt-3 space-y-4">
                <div>
                  <dt class="text-sm font-medium text-muted">
                    Federal Ordinary Taxable Income
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(federalOrdinaryIncome) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    FICA Taxable Income
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(ficaWage) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    State Taxable Income
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(massachusettsTaxableIncome) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Long Term Gains Taxable Income
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(longTermTaxableIncome) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Net Investment Taxable Income
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(netInvestmentTaxableIncome) }}
                  </dd>
                </div>
              </dl>
            </div>

            <div class="border-t border-default pt-5">
              <h3 class="text-sm font-semibold text-highlighted">
                Taxes
              </h3>

              <dl class="mt-3 space-y-4">
                <div>
                  <dt class="text-sm font-medium text-muted">
                    Federal Tax
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(federalTax) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    FICA Tax
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(ficaTax) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    State Tax
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(stateTax) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Long Term Gains Tax
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(longTermTax) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Net Investment Income Tax
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(netInvestmentIncomeTax) }}
                  </dd>
                </div>

                <div class="border-t border-default pt-4">
                  <dt class="text-sm font-medium text-muted">
                    Total Tax
                  </dt>
                  <dd class="mt-1 text-2xl font-semibold text-highlighted">
                    {{ formatCurrency(totalTax) }}
                  </dd>
                </div>
              </dl>
            </div>

            <div class="border-t border-default pt-5">
              <h3 class="text-sm font-semibold text-highlighted">
                Retirement and HSA Contributions
              </h3>

              <dl class="mt-3 space-y-4">
                <div>
                  <dt class="text-sm font-medium text-muted">
                    401(k) (Employee)
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(adjustments.retirement401kContributions ?? 0) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    HSA Contributions (Employee)
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(hsaEmployeeContributionAmount) }}
                  </dd>
                </div>
              </dl>
            </div>

            <div class="border-t border-default pt-5">
              <h3 class="text-sm font-semibold text-highlighted">
                Summary
              </h3>

              <dl class="mt-3 space-y-4">
                <div>
                  <dt class="text-sm font-medium text-muted">
                    Effective Tax Rate
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatPercent(effectiveTaxRate) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Marginal Tax Rate (Wage Income)
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatPercent(marginalTaxRateWageIncome) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Marginal Tax Rate (Investment Income)
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatPercent(marginalTaxRateInvestmentIncome) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Marginal Tax Rate (Long Term & Qualified Income)
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatPercent(marginalTaxRateLongTermIncome) }}
                  </dd>
                </div>

                <div class="border-t border-default pt-4">
                  <dt class="text-sm font-medium text-muted">
                    Net Income
                  </dt>
                  <dd class="mt-1 text-2xl font-semibold text-highlighted">
                    {{ formatCurrency(netIncome) }}
                  </dd>
                </div>

                <div>
                  <dt class="text-sm font-medium text-muted">
                    Net Income (minus Investment Income)
                  </dt>
                  <dd class="mt-1 text-xl font-semibold text-highlighted">
                    {{ formatCurrency(netIncomeMinusInvestmentIncome) }}
                  </dd>
                </div>
              </dl>
            </div>
          </div>
        </section>
      </aside>
    </div>
  </UContainer>
</template>
