# Falculator: Advanced Financial Simulation System

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Technical Architecture](#technical-architecture)
3. [Core Capabilities](#core-capabilities)
4. [Financial Item Framework](#financial-item-framework)
5. [Simulation Engine](#simulation-engine)
6. [Configuration System](#configuration-system)
7. [User Interface](#user-interface)
8. [Development Information](#development-information)
9. [Use Cases & Examples](#use-cases--examples)
10. [Roadmap & Future Features](#roadmap--future-features)

## Executive Summary

Falculator is a sophisticated, Windows Forms-based financial simulation application built on .NET 8. It provides unprecedented flexibility for modeling complex financial scenarios through a rule-based, configuration-driven approach. The application excels at simulating wealth accumulation, investment strategies, debt management, and retirement planning with precise temporal control and interdependent financial relationships.

### Key Differentiators
- **Rule-Based Automation**: Complex trigger systems enable sophisticated financial decision chains
- **Temporal Precision**: Day-level simulation granularity with configurable step increments
- **Relationship Modeling**: Financial items can interact, make payments, and trigger each other
- **Template-Driven**: Pre-configured templates accelerate common scenario setup
- **Real-Time Feedback**: Live progress tracking and debugging during simulations

## Technical Architecture

### Application Structure
```
Falculator/
+-- Program.cs              # Entry point with splash screen management
+-- Form1.cs                # Main UI with tabbed interface
+-- Models/                 # Core data models
|   +-- Config.cs          # Simulation configuration
|   +-- FinancialItem.cs   # Complex financial instruments
|   +-- SimFrame.cs        # Single simulation time step
|   +-- AmountFreq.cs      # Amount/frequency calculations
|   +-- ...
+-- Services/              # Business logic
|   +-- SimulatorEngine.cs # Core simulation processor
|   +-- DebugLogger.cs     # UI-integrated logging
+-- Templates/             # Pre-built financial item templates
+-- Examples/              # Complete scenario examples
```

### Technology Stack
- **Framework**: .NET 8 Windows Forms
- **Data**: JSON-based configuration with System.Text.Json
- **UI**: PropertyGrid-driven configuration, tabbed interface
- **Architecture**: Event-driven with real-time UI updates

## Core Capabilities

### 1. Financial Item System
Each financial item represents a financial entity with comprehensive capabilities:

#### Identity & Classification
- **Tagging System**: Flexible categorization for rule-based referencing
- **Type Classification**: Income, Savings, Asset, or Liability designations  
- **Metadata**: Currency, liquidity status, descriptions, and custom attributes

#### Cash Flow Management
- **Multi-frequency Operations**: Daily, weekly, monthly, or annual transactions
- **Percentage-based Calculations**: Revenue/expenditure as fixed amounts or percentages
- **Compound Interest**: Sophisticated interest calculations with configurable compounding

#### Lifecycle Control
- **Temporal Activation**: Schedule items to activate on specific dates or conditions
- **Conditional Logic**: Complex trigger systems for activation/deactivation
- **Processing Order**: Explicit evaluation sequence for handling dependencies

#### Relationship Network
- **Payment Systems**: Items can make payments to other items
- **Cross-item Triggers**: Items can activate/deactivate based on other items' states
- **Conditional Chains**: Create cascading sequences of financial events

### 2. Advanced Trigger System

The trigger system enables sophisticated financial automation:

```csharp
// Example: Property investment trigger
TriggerConditions {
    RequireAll: true,
    LiquidAssetsEnabled: true,
    LiquidAssetsAmountValue: 50000, // $50k buffer + deposit
    TagRules: [
        {
            Tags: ["existing-properties"],
            MatchAll: false, // Any existing property
            MatchOnTrue: true, // Must be active
            RequireCount: 1 // At least one active
        }
    ]
}
```

#### Trigger Types
- **Date-based**: Activate on specific dates or age milestones
- **Balance-based**: Respond to liquid asset thresholds
- **Income-based**: Trigger on passive income levels
- **Tag-based**: Conditional activation based on other tagged items
- **Multi-factor**: Combine multiple conditions with AND/OR logic

#### Trigger Modes
- **One-time**: Fire once and disable
- **Recurring**: Re-evaluate continuously
- **Conditional Chains**: Sequential activation patterns

## Financial Item Framework

### Core Properties

#### Financial Classification
```json
{
  "Type": 1, // 0=Income, 1=Savings, 2=Asset, 3=Liability
  "Name": "Investment Property 1",
  "Description": "First rental property",
  "Tags": ["property", "investment", "rental"],
  "Currency": "AUD",
  "IsLiquidAsset": false
}
```

#### Cash Flow Configuration
```json
{
  "Revenue": {
    "Amount": 2400, // $2400/month rental
    "Frequency": 3  // Monthly
  },
  "Expenditure": {
    "Amount": 3200, // $3200/month interest + expenses
    "Frequency": 3
  },
  "Interest": {
    "Amount": 4.5, // 4.5% annual interest
    "Frequency": 4 // Annual compounding
  }
}
```

#### Payment Networks
```json
{
  "Events": [
    {
      "Name": "Main Savings",
      "Amount": 100, // Percentage of revenue
      "IsPercentage": true,
      "Frequency": 3, // Monthly transfer
      "Tags": ["savings"] // Target by tag
    }
  ]
}
```

### Relationship System

Financial items can create complex interdependencies:

1. **Payment Flows**: Automatic transfers between accounts
2. **Trigger Dependencies**: Items can activate others based on conditions
3. **Tag-based References**: Group operations and conditional logic
4. **Evaluation Ordering**: Control processing sequence for dependencies

## Simulation Engine

### Processing Architecture

The simulation engine (`SimulatorEngine.cs`) processes time-stepped simulations:

```csharp
public class SimulatorEngine
{
    // Core simulation loop
    public bool SimulationStep()
    {
        // 1. Create simulation frame for current time step
        // 2. Process all active financial items
        // 3. Calculate revenues, expenditures, interest
        // 4. Execute payment networks
        // 5. Evaluate trigger conditions
        // 6. Update UI progress
        // 7. Store frame in simulation history
    }
}
```

### Simulation Flow

1. **Initialization**: Deep copy configuration to prevent mutation
2. **Time Stepping**: Advance by configured increment (daily/weekly/monthly/annual)
3. **Frame Processing**: For each time step:
   - Calculate due payments and interest
   - Process revenue and expenditure
   - Execute inter-item payments
   - Evaluate trigger conditions
   - Update item states
4. **Progress Tracking**: Real-time UI updates with cancellation support
5. **Data Collection**: Store complete simulation history for analysis

### Performance Features

- **Configurable Granularity**: Balance precision vs. performance
- **Progress Visualization**: Real-time progress bar and status updates
- **Cancellation Support**: User can halt long-running simulations
- **Memory Efficient**: Reuse objects where possible
- **UI Responsiveness**: Periodic UI refresh during processing

## Configuration System

### User Settings
```json
{
  "BirthDate": "1985-01-01", // For age-based triggers
  "UserName": "John Doe",
  "SimulationName": "Retirement Plan 2024",
  "Description": "Conservative investment strategy",
  "StepIncrement": "Monthly", // Daily/Weekly/Monthly/Annual
  "YearsToSim": 30, // Simulation duration
  "RollingBackup": true,
  "BackupCount": 10
}
```

### Backup System
- **Automatic Backups**: Rolling backups prevent configuration loss
- **Version Control**: Track save count and modification dates
- **Recovery**: Easy restoration from previous versions

## User Interface

### Main Interface Components

#### Tabbed Layout
1. **Configuration Tab**: PropertyGrid for simulation settings
2. **Financial Items Tab**: Manage individual financial instruments  
3. **Results Tab**: Simulation output and analysis
4. **Debug Tab**: Real-time logging and troubleshooting

#### Property Grid Integration
- **Categorized Properties**: Logical grouping of configuration options
- **Custom Type Converters**: User-friendly display of complex types
- **Validation**: Real-time input validation and error handling
- **Help Integration**: Contextual descriptions and examples

#### Status & Progress
- **Progress Bar**: Visual simulation progress with cancellation
- **Status Labels**: Real-time status updates and messages
- **Debug Output**: Integrated logging to RichTextBox control

### Template System

#### Pre-built Templates
- **Savings Account**: Basic interest-bearing savings
- **Investment Properties**: Rental property with loan structures
- **Superannuation**: Age-restricted retirement accounts
- **Income Streams**: Salary, business income, passive income

#### Custom Templates
Users can save their own financial items as templates for reuse across scenarios.

## Development Information

### Build & Run
```bash
cd Falculator
dotnet build                    # Build project
dotnet run                      # Run application
dotnet build --configuration Release  # Release build
```

### Code Architecture
- **Models**: Data structures with PropertyGrid attributes
- **Services**: Business logic separation from UI
- **Event-Driven**: Responsive UI with background processing
- **JSON Serialization**: Human-readable configuration files
- **Type Safety**: Nullable reference types and comprehensive validation

### Key Design Patterns
- **Deep Cloning**: Simulation uses immutable config copies
- **Observer Pattern**: UI updates from simulation events
- **Strategy Pattern**: Configurable calculation frequencies
- **Template Method**: Consistent simulation frame processing

## Use Cases & Examples

### Example 1: Basic Wealth Accumulation
```json
{
  "salary": {
    "Type": 0, // Income
    "Revenue": {"Amount": 80000, "Frequency": 4}, // $80k annually
    "Events": [
      {"Name": "Living Expenses", "Amount": 60000, "IsPercentage": false},
      {"Name": "Savings", "Amount": 20000, "IsPercentage": false}
    ]
  },
  "savings": {
    "Type": 1, // Savings
    "Interest": {"Amount": 4.5, "Frequency": 4}, // 4.5% annual
    "IsLiquidAsset": true
  }
}
```

### Example 2: Negative Gearing Property Portfolio

This sophisticated example demonstrates Falculator's ability to model complex, interdependent financial strategies:

```json
{
  "mainSavings": {
    "Type": 1,
    "Name": "Main Savings Buffer",
    "Value": 100000,
    "IsLiquidAsset": true,
    "Interest": {"Amount": 3.5, "Frequency": 4}
  },
  "property1": {
    "Type": 2,
    "Name": "Investment Property 1", 
    "IsActive": false,
    "Triggers": {
      "LiquidAssetsEnabled": true,
      "LiquidAssetsAmountValue": 80000, // $50k buffer + $30k deposit
      "LiquidAssetsGreaterThan": true
    },
    "Value": 600000,
    "Revenue": {"Amount": 2400, "Frequency": 3}, // $2400/month rent
    "Expenditure": {"Amount": 3200, "Frequency": 3}, // Interest + expenses
    "Events": [
      {"Name": "Main Savings", "Amount": 2400, "IsPercentage": false}
    ]
  },
  "property2": {
    "Type": 2, 
    "Name": "Investment Property 2",
    "IsActive": false,
    "Triggers": {
      "RequireAll": true,
      "LiquidAssetsEnabled": true,
      "LiquidAssetsAmountValue": 80000,
      "TagRules": [
        {
          "Tags": ["property1"],
          "MatchOnTrue": true, // Property 1 must be active
          "RequireCount": 1
        }
      ]
    }
    // Similar structure to property1
  }
}
```

**Scenario Flow:**
1. **Initial State**: $100k in savings, no properties active
2. **Property 1 Trigger**: When savings > $80k, Property 1 activates
3. **Cash Flow**: Property 1 generates $2400/month rent, costs $3200/month (negative $800)
4. **Portfolio Growth**: Property 1 pays rent to savings; negative gearing reduces taxes
5. **Property 2 Trigger**: When savings recover to $80k AND Property 1 is active, Property 2 activates
6. **Scaling**: Pattern repeats for additional properties
7. **Exit Strategy**: When total assets exceed target, liquidation triggers activate

### Example 3: Retirement Transition
```json
{
  "superannuation": {
    "Type": 1,
    "Triggers": {
      "AgeEnabled": true,
      "AgeGreaterThan": true,
      "AgeValue": 60 // Access at age 60
    },
    "Value": 500000,
    "Interest": {"Amount": 6.5, "Frequency": 4}
  },
  "retirementIncome": {
    "Type": 0,
    "IsActive": false,
    "Triggers": {
      "AgeEnabled": true,
      "AgeValue": 65 // Start drawdown at 65
    },
    "Revenue": {"Amount": 40000, "Frequency": 4} // $40k annual drawdown
  }
}
```

## Current Features

### Implemented
- [x] **Complete Financial Item Framework**: Full-featured financial modeling
- [x] **Advanced Trigger System**: Complex conditional activation
- [x] **Simulation Engine**: Multi-frequency time-step processing
- [x] **Template System**: Pre-built and custom templates
- [x] **Progress Tracking**: Real-time simulation feedback
- [x] **Data Export**: JSON configuration save/load
- [x] **Debug Logging**: Comprehensive troubleshooting tools
- [x] **Backup System**: Automatic rolling backups

### User Interface
- [x] **PropertyGrid Integration**: Rich configuration editing
- [x] **Tabbed Interface**: Organized workflow
- [x] **Splash Screen**: Professional application startup
- [x] **Status Updates**: Real-time progress and messaging

## Roadmap & Future Features

### MVP Priorities (In Development)
- [IN PROGRESS] **Interactive Graphing**: Visual representation of financial trajectories
- [IN PROGRESS] **Excel Export**: Export simulation results to spreadsheets
- [IN PROGRESS] **PDF Reporting**: Professional report generation with charts
- [IN PROGRESS] **Event Markers**: Visual timeline indicators for key events

### Advanced Features (Post-MVP)
- [PLANNED] **Scenario Testing**: CSV-based economic stress testing
  - Import economic conditions by time period
  - Model inflation, interest rate changes, market downturns
  - Forward projection from final CSV line
  - Multiple scenario comparison analysis

- [PLANNED] **Enhanced Analytics**:
  - Monte Carlo simulations
  - Sensitivity analysis
  - Risk assessment metrics
  - Goal achievement probability

- [PLANNED] **User Experience**:
  - Wizard-based setup for common scenarios
  - Enhanced template library
  - Collaboration features
  - Mobile companion app

### Professional Features
- [PLANNED] **Client Management**: Multi-client scenario management
- [PLANNED] **Compliance Reporting**: Professional financial planning reports
- [PLANNED] **API Integration**: External data feeds for real-time rates
- [PLANNED] **Advanced Export**: Integration with professional planning software

## Licensing Strategy

### Proposed Tiers
- **Personal (Free)**: 3 financial items maximum
- **Professional ($89-149)**: Unlimited items, full feature set
- **Enterprise**: Custom pricing for financial planning firms

### Compliance
- Professional edition enforced through software auditing
- Commercial use detection drives professional tier adoption

---

## Technical Notes

### Data Persistence
- **JSON Configuration**: Human-readable, version-controllable
- **Deep Cloning**: Simulation isolation prevents config corruption  
- **Backup Integration**: Automatic versioning with rollback capability

### Performance Characteristics
- **Memory Efficient**: Reuse simulation frames where possible
- **Responsive UI**: Background processing with progress feedback
- **Scalable**: Handles complex scenarios with hundreds of financial items
- **Cancellation**: User can interrupt long-running simulations

### Extensibility
- **Plugin Architecture**: Modular financial item types
- **Custom Calculations**: Extensible interest and payment calculations
- **Template System**: User-defined templates and scenarios
- **Import/Export**: JSON-based configuration sharing

This comprehensive financial simulation system provides unprecedented flexibility for modeling complex financial scenarios, making it valuable for individuals managing personal finances and professionals designing sophisticated client strategies.