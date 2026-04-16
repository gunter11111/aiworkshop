# ABL Business Entity Architecture Pattern

## Overview

The Business Entity pattern provides a standardized, maintainable approach to data access in OpenEdge ABL applications. It separates UI logic from database operations through a layered architecture that promotes reusability, testability, and consistency across the application.

## Architecture Layers

### 1. UI Layer (Windows/Forms)
- **Responsibility**: User interaction and presentation
- **Access**: Never directly accesses database tables
- **Communication**: Calls Business Entity methods with datasets

### 2. Business Entity Layer
- **Responsibility**: Data access, business rules, validation
- **Inheritance**: Extends `OpenEdge.BusinessLogic.BusinessEntity`
- **Management**: Instantiated through EntityFactory (singleton pattern)

### 3. Database Layer
- **Responsibility**: Persistent storage
- **Access**: Only through data-sources attached to business entities

## Key Components

### EntityFactory (Singleton Pattern)

**Purpose**: Centralized management of business entity lifecycle

**Pattern**:
```abl
CLASS business.EntityFactory:
    VAR PRIVATE STATIC EntityFactory objInstance.
    VAR PRIVATE CustomerEntity objCustomerEntityInstance.
    
    CONSTRUCTOR PRIVATE EntityFactory():
    END CONSTRUCTOR.
    
    METHOD PUBLIC STATIC EntityFactory GetInstance():
        IF objInstance = ? THEN
            objInstance = NEW EntityFactory().
        RETURN objInstance.
    END METHOD.
    
    METHOD PUBLIC CustomerEntity GetCustomerEntity():
        IF objCustomerEntityInstance = ? THEN
            objCustomerEntityInstance = NEW CustomerEntity().
        RETURN objCustomerEntityInstance.
    END METHOD.
END CLASS.
```

### Dataset Definition (.i Include Files)

**Pattern**:
```abl
DEFINE TEMP-TABLE ttCustomer BEFORE-TABLE bttCustomer
    FIELD CustNum AS INTEGER INITIAL "0" LABEL "Cust Num"
    FIELD Name AS CHARACTER LABEL "Name"
    INDEX CustNum IS PRIMARY UNIQUE CustNum ASCENDING.

DEFINE DATASET dsCustomer FOR ttCustomer.
```

**Key Points**:
- `BEFORE-TABLE` enables change tracking for updates
- Temp-table fields match database table structure
- Primary index mirrors database primary key

### Business Entity Class

**Pattern**:
```abl
CLASS business.CustomerEntity INHERITS BusinessEntity USE-WIDGET-POOL:
    {business/CustomerDataset.i}
    DEFINE DATA-SOURCE srcCustomer FOR Customer.
    
    CONSTRUCTOR PUBLIC CustomerEntity():
        SUPER(DATASET dsCustomer:HANDLE).
        VAR HANDLE[1] hDataSourceArray = DATA-SOURCE srcCustomer:HANDLE.
        VAR CHARACTER[1] cSkipListArray = [""].
        THIS-OBJECT:ProDataSource = hDataSourceArray.
        THIS-OBJECT:SkipList = cSkipListArray.
    END CONSTRUCTOR.
END CLASS.
```

## Standard CRUD Operations

### Read
```abl
METHOD PUBLIC LOGICAL GetCustomerByNumber(INPUT ipiCustNum AS INTEGER,
                                          OUTPUT DATASET dsCustomer):
    VAR CHARACTER cFilter = "WHERE Customer.CustNum = " + STRING(ipiCustNum).
    THIS-OBJECT:ReadData(cFilter).
    RETURN CAN-FIND(FIRST ttCustomer).
END METHOD.
```

### Update (from UI)
```abl
lFound = objEntity:GetCustomerByNumber(iNum, OUTPUT DATASET dsCustomer).
IF lFound THEN DO:
    FIND FIRST ttCustomer.
    TEMP-TABLE ttCustomer:TRACKING-CHANGES = TRUE.
    ttCustomer.Name = "Updated Name".
    isValid = objEntity:ValidateCustomer(DATASET dsCustomer BY-REFERENCE, OUTPUT cError).
    IF isValid THEN
        objEntity:UpdateCustomer(DATASET dsCustomer BY-REFERENCE).
END.
```

## Common Pitfalls

1. **BY-REFERENCE on OUTPUT DATASET for Read** - Use `OUTPUT DATASET` without BY-REFERENCE
2. **Forgetting Change Tracking** - Always set `TRACKING-CHANGES = TRUE` before modifications
3. **Direct Database Access from UI** - Always use business entity methods

## References

- `src/business/CustomerEntity.cls`
- `src/business/EntityFactory.cls`
- `src/CustomerWin.w`
