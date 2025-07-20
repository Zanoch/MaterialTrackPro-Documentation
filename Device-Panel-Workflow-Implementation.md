# Device-Panel Workflow Implementation

## Overview

The Device-Panel application implements comprehensive material tracking workflows for tea processing operations within the MaterialTrackPro (MTP) platform. Operating primarily at the SEEDUWA processing site, it manages material acceptance, recipe-based processing, and dispatch operations through hardware integration and API communication with Central-Handler, which serves as a proxy to AWS-Manager for order management operations.

## Workflow Architecture

### Multi-Site Material Flow Architecture

The MTP platform operates across two primary sites with distinct operational roles:

**SEEDUWA Site (Processing & Dispatch)**:
- Material processing and transformation operations
- Device-Panel hardware integration for physical operations
- Material allocation and dispatch preparation
- Central-Handler database for processing records

**GRANDPASS Site (Planning & Receiving)**:
- Order planning and request management
- Shipment receiving and acceptance operations
- AWS-Manager database for order management
- Admin-Panel for parcel refinement operations

### Communication Architecture

**Device-Panel Integration**:
- Device-Panel ↔ Central-Handler ↔ AWS-Manager (proxy communication)
- Central-Handler serves as proxy for AWS-Manager operations
- Hardware integration handled locally at SEEDUWA

**Admin-Panel Integration**:
- Admin-Panel ↔ AWS-Manager ↔ Central-Handler (reverse proxy)
- AWS-Manager serves as proxy for Central-Handler operations
- Parcel refinement operations handled by Admin-Panel

### Material Flow Patterns

The Device-Panel implements material workflows based on allocation table patterns that define processing relationships and dispatch destinations. The allocation system follows an output-based design where materials are allocated to specific processing outputs or dispatch destinations.

#### Current Allocation Relationships

**Processing Input Allocations** (Current Implementation):
- `tealine_allocate_blendsheet`: Tealine materials → Blendsheet processing
- `blendbalance_allocate_blendsheet`: Blendbalance materials → Blendsheet processing

**Planned Processing Input Allocations**:
- `blendsheet_allocate_flavorsheet`: Blendsheet materials → Flavorsheet processing
- `herbline_allocate_flavorsheet`: Herbline materials → Flavorsheet processing

**Serial Dispatch Allocations** (Current Implementation):
- **Tealine Flow**: `tealine_allocate_shipment` → `tealine_allocate_parcel` (⚠️ **future removal planned**)
- **Blendsheet Flow**: `blendsheet_allocate_shipment` → `blendsheet_allocate_parcel`

**Planned Serial Dispatch Allocations**:
- **Flavorsheet Flow**: `flavorsheet_allocate_shipment` → `flavorsheet_allocate_parcel`

#### Material Classification by Processing Role

**1. Pure Processing Inputs**:
- **Blendbalance**: Accept → `blendbalance_allocate_blendsheet` → Ingredient use only (no dispatch by design)

**2. Multi-Purpose Materials**:
- **Tealine**: Accept → `tealine_allocate_blendsheet` (processing input) + ~~`tealine_allocate_shipment`~~ (dispatch, future removal)
- **Blendsheet**: Process → Accept → `blendsheet_allocate_flavorsheet` (planned processing input) + `blendsheet_allocate_shipment` (dispatch)

**3. Future Implementation Materials**:
- **Herbline**: Accept → `herbline_allocate_flavorsheet` (planned processing input) → Dispatch fate uncertain
- **Flavorsheet**: Process → Accept → `flavorsheet_allocate_shipment` (planned dispatch)

#### Complete Material Flow Architecture

**Processing Chain**:
```
Tealine + Blendbalance → tealine_allocate_blendsheet + blendbalance_allocate_blendsheet → Blendsheet
Blendsheet + Herbline → blendsheet_allocate_flavorsheet + herbline_allocate_flavorsheet → Flavorsheet
```

**Serial Dispatch Chain**:
```
Tealine → tealine_allocate_shipment → tealine_allocate_parcel (future removal)
Blendsheet → blendsheet_allocate_shipment → blendsheet_allocate_parcel  
Flavorsheet → flavorsheet_allocate_shipment → flavorsheet_allocate_parcel (planned)
```

### Serial Allocation Architecture

The allocation system uses a **serial flow pattern** where parcel allocations are refinements of existing shipment allocations:

**Design Principle**: `{material}_allocate_shipment` → `{material}_allocate_parcel`

**Serial Flow Constraints**:
- Parcel allocation requires existing shipment allocation (foreign key dependency)
- Parcel allocation refines/subdivides shipment quantities for production scheduling
- Total parcel allocations cannot exceed shipment allocation quantities
- Device-Panel handles shipment allocations only
- Admin-Panel handles parcel refinement operations

**Current Allocation Support**:
- ✅ **Tealine**: Shipment allocation + Blendsheet processing allocation
- ✅ **Blendsheet**: Shipment allocation + Flavorsheet processing allocation (planned)
- ✅ **Blendbalance**: Blendsheet processing allocation only
- ⏳ **Herbline**: Flavorsheet processing allocation (planned)
- ⏳ **Flavorsheet**: Shipment allocation (planned)

## Shipment Workflow Integration

### Device-Panel Shipment Operations

The Device-Panel manages the physical shipment operations at SEEDUWA with the following workflow:

#### Order Retrieval and Material Readiness

Device-Panel retrieves orders ready to ship from AWS-Manager via Central-Handler proxy:

```javascript
// From DispatchTransitSelect.jsx - Orders ready to ship retrieval
const setupResources = async () => {
  try {
    const transitList = await fetch(`${window.base_url}/app/shipment`).then(async (res) => {
      if (!res.ok)
        throw new Error(res.status === 500 ? "Internal Server Error" : await res.text());
      return res.headers.get("content-type").startsWith("application/json")
        ? await res.json()
        : await res.text();
    });
    updateTransitList(
      transitList
        .filter(({ quantity, cart_qty }) => cart_qty < quantity)
    );
    updateResources({ status: "ready" });
  } catch (e) {
    updateResources({
      status: "error",
      error: new SelectPanelError("Bag Acceptance Failed", e.message),
    });
  }
};
```

#### Material Scanning and Shipment Allocation

Device-Panel performs material scanning and creates shipment allocation records:

```javascript
// Shipment allocation creation (Device-Panel scope)
await fetch(`${window.base_url}/app/${table}/record/shipment`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    request_code,
    shipment_code,
    barcode,
    amount: required,
  }),
});
```

#### Dispatch Event Creation

Device-Panel creates dispatch events and coordinates with multi-site operations:

```javascript
// From DispatchTransitCart.jsx - Dispatch event creation
const response = await fetch(`${window.base_url}/app/shipment/event`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(
    shipments.map(({ request_code, shipment_code }) => ({
      request_code,
      shipment_code,
      shipment_vehicle: value,
    }))
  ),
});
```

### Multi-Site Coordination

**SEEDUWA Operations** (Device-Panel Focus):
- Physical material scanning and validation
- Shipment allocation record creation (`*_allocate_shipment`)
- Order retrieval and dispatch event creation via Central-Handler proxy
- Hardware integration for weighing, scanning, and printing

**GRANDPASS Operations** (Minimal Context):
- Order management and shipment status tracking
- Shipment receiving and acceptance verification  
- Parcel refinement operations via Admin-Panel

## State Management Architecture

### React Context Pattern

The application uses TransitCart context for cross-component state sharing in dispatch workflows:


### Context-based State Management

```javascript
// From TransitCart.jsx - Context implementation
const TransitCartContext = createContext();

export function useTransitCart() {
  return useContext(TransitCartContext);
}

export default function TransitCartProvider({ children }) {
  const [cart, updateCart] = useState([]);

  const addToCart = (item) => {
    updateCart([...cart, item]);
  };

  const removeFromCart = (item) => {
    updateCart(cart.filter((i) => i !== item));
  };

  const clearCart = () => {
    updateCart([]);
  };

  return (
    <TransitCartContext.Provider value={{ cart, addToCart, removeFromCart, clearCart }}>
      {children}
    </TransitCartContext.Provider>
  );
}
```

### Form State Management

Form data handling with quantity calculations in dispatch workflows:

```javascript
// From DispatchTransitSelect.jsx - QuantityInput form management
function QuantityInput() {
  const { formData, updateFormData } = useSelectPanelContext();

  useEffect(() => {
    let shipQty = 0;
    if (formData?.selectValue) {
      const { quantity, cart_qty } = formData.selectValue;
      shipQty = quantity - cart_qty;
    }
    updateFormData((formData) => ({
      ...formData,
      shipQty,
    }));
  }, [formData?.selectValue]);

  const onQuantityChange = (event) => {
    updateFormData((formData) => ({
      ...formData,
      shipQty: event.target.value,
    }));
  };
```

### Hardware State Integration

Real-time hardware data integration with React state management:

```javascript
// From AcceptTealineWeight.jsx - Hardware listeners setup
useEffect(() => {
  if (resources.status === "ready") {
    const listeners = [];
    if (!locked) {
      listeners.push(
        app.serialRx("loadcell", (_, data) => {
          updateLoadCellValue(data);
        })
      );
    }
    listeners.push(
      app.serialRx("button", (_, data) => {
        if (data.mode === "BAG_ACCEPT") onAddRecordClick();
      })
    );
    return () => {
      listeners.forEach((cleanFn) => {
        cleanFn();
      });
    };
  }
});
```

## Accept Workflows

### Pattern: Two-Stage Accept Process

All Accept workflows follow a consistent two-stage pattern optimized for physical operations:

1. **Selection Stage**: Choose items/batches and configure operational parameters
2. **Weight Recording Stage**: Physical weighing, barcode generation, location assignment, and record creation

### Tealine Accept Implementation

#### Selection Phase (AcceptTealineSelect.jsx)

The selection phase handles item selection and parameter configuration:

```javascript
// From AcceptTealineSelect.jsx - Resource initialization
const setupResources = async () => {
  try {
    const tealineList = await fetch(`${window.base_url}/app/tealine`).then(
      async (res) => {
        if (!res.ok)
          throw new Error(res.status === 500 ? "Internal Server Error" : await res.text());
        return res.headers.get("content-type").startsWith("application/json")
          ? await res.json()
          : await res.text();
      }
    );
    updateTealineList(tealineList);
    updateResources({ status: "ready" });
  } catch (e) {
    updateResources({
      status: "error",
      error: new SelectPanelError("Tealine Acceptance Failed", e.message),
    });
  }
};
```

#### Weight Recording Phase (AcceptTealineWeight.jsx)

The weight recording phase integrates hardware devices for data capture:

```javascript
// From AcceptTealineWeight.jsx - Record creation
const onAddRecordClick = () => {
  let errorOccurred = false;
  if (locationIndex === "") {
    errorOccurred = true;
    setLocationError("Please select a store location.");
  }
  if (tareWeightValue === "" || Number.isNaN(Number(tareWeightValue))) {
    errorOccurred = true;
    setTareWeightError("Please enter a numeric value.");
  }
  if (errorOccurred) return;
  
  setAddRecordBusy(true);

  fetch(`${window.base_url}/app/tealine/record`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      item_code,
      created_ts,
      store_location: locationList[locationIndex].store_location,
      gross_weight: loadCellValue.reading,
      bag_weight: tareWeightValue,
    }),
  })
    .then((res) => {
      if (res.ok) return res.json();
      throw new Error(res.status === 500 ? "Internal Server Error" : res.text());
    })
    .then(({ id, received_ts, barcode }) => {
      // State update logic
      const { record_list, ...tealineItem } = selectedTealine;
      updateSelectedTealine({
        ...tealineItem,
        record_list: record_list.concat([{
          id,
          barcode,
          received_ts: Number(received_ts),
          store_location: locationList[locationIndex].store_location,
          gross_weight: loadCellValue.reading,
          bag_weight: tareWeightValue,
        }]),
      });
      
      // Automatic label printing
      app.serialTx("printer", {
        "Item Code": selectedTealine.item_code,
        Garden: selectedTealine.garden,
        Broker: selectedTealine.broker,
        "Received At": DateTime.fromMillis(Number(received_ts)).toFormat("hh:mm:ss a, dd/LL/yyyy"),
        Location: locationList[locationIndex].store_location,
        "Gross Weight": loadCellValue.reading,
        "Bag Weight": tareWeightValue,
        "Net Weight": [loadCellValue.reading, tareWeightValue]
          .map((v) => Math.round(v * 100))
          .reduce((a, b) => a - b) / 100,
        "Bag No": `${id}`,
        barcode: `tealine:${barcode}`,
      });
    })
    .finally(() => {
      setAddRecordBusy(false);
    });
};
```


## Process Workflows

### Pattern: Recipe-Based Processing with Optional Batch Configuration

Process workflows implement recipe-driven material transformation with optional batch configuration:

1. **Selection Stage**: Choose recipes and optionally configure batch parameters
2. **Scanning Stage**: Scan ingredient barcodes for recipe fulfillment with allocation tracking

### Blendsheet Process Implementation

#### Recipe Selection with Optional Batch Configuration (ProcessBlendsheetSelect.jsx)

Recipe selection with optional batch size configuration:

```javascript
// From ProcessBlendsheetSelect.jsx - Recipe selection with optional batch configuration
const onSelectPanelStartClick = async ({ selectValue, batchSize }) => {
  if (!selectValue) return;
  
  // Optional batch size validation - if provided, must be valid
  if (batchSize && Number.isNaN(Number.parseInt(batchSize))) return;
  
  const response = await fetch(`${window.base_url}/app/blendsheet/batch`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      blendsheet_no: selectValue.blendsheet_no,
    }),
  });
  
  const { item_code, created_ts } = await response.json();
  
  history.push("/blendsheet/scan", { 
    item_code, 
    created_ts, 
    // Optional batch size parameter
    ...(batchSize && { batchSize: Number.parseInt(batchSize) })
  });
};
```

#### Ingredient Scanning with Allocation (ProcessBlendsheetScan.jsx)

The scanning phase implements allocation-based ingredient tracking:

```javascript
// From ProcessBlendsheetScan.jsx - Barcode scanning with allocation
const onBarcodeSubmit = async (ev) => {
  ev.preventDefault();
  const scannedBarcode = ev.target.barcode.value;
  
  const scannedRecord = await fetch(
    `${window.base_url}/app/scan?table_name=${table}&barcode=${scannedBarcode}`
  ).then(async (res) => {
    if (res.ok) return res.json();
    throw new Error("Invalid barcode or material not found");
  });
  
  if (!["ACCEPTED", "IN_PROCESS"].includes(scannedRecord.status)) {
    throw new Error("Material not available for processing");
  }
  
  // Allocation-based ingredient assignment
  await fetch(`${window.base_url}/app/${table}/record/blendsheet`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      barcode: scannedBarcode,
      item_code,
      created_ts: Number(created_ts),
      amount: required,
    }),
  });
};
```

## Dispatch Workflows

### Pattern: Three-Stage Dispatch Process

Dispatch workflows manage material-out operations with comprehensive allocation tracking:

1. **Transit Selection**: Retrieve orders ready to ship and configure quantities
2. **Material Scanning**: Scan finished goods for shipment allocation
3. **Cart Management**: Review and confirm shipments with dispatch details

### Current Dispatch Capability

**Current Dispatch Support**:
- ✅ **Blendsheet**: Complete Process → Accept → Dispatch workflow (`blendsheet_allocate_shipment`)
- ⚠️ **Tealine**: Accept → Dispatch workflow (`tealine_allocate_shipment`) - **future removal planned**

**Planned Dispatch Implementation**:
- ⏳ **Flavorsheet**: Process → Accept → Dispatch (`flavorsheet_allocate_shipment`)

### Dispatch Transit Implementation

#### Orders Ready to Ship Selection (DispatchTransitSelect.jsx)

Integration with orders ready to ship through Central-Handler proxy:

```javascript
// From DispatchTransitSelect.jsx - Orders ready to ship with cart filtering
updateTransitList(
  transitList
    .map(({ request_code, shipment_code, ...rest }) => ({
      request_code,
      shipment_code,
      ...rest,
      cart_qty: cart
        .filter((item) =>
          Object.entries({ request_code, shipment_code }).every(
            ([key, value]) => item[key] === value
          )
        )
        .reduce((sum, { ship_qty }) => sum + ship_qty, 0),
    }))
    .filter(({ quantity, cart_qty }) => cart_qty < quantity)
);
```

#### Cart Management and Dispatch (DispatchTransitCart.jsx)

Cart management with comprehensive dispatch processing:

```javascript
// From DispatchTransitCart.jsx - Cart state management with TransitCart context
const { cart, removeFromCart, clearCart } = useTransitCart();

// Shipment allocation processing
await Promise.all(
  shipments.flatMap(({ request_code, shipment_code, records }) =>
    records.map(async ({ table, barcode, required }) => {
      const response = await fetch(`${window.base_url}/app/${table}/record/shipment`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          request_code,
          shipment_code,
          barcode,
          amount: required,
        }),
      });
      if (!response.ok) throw new Error(await response.text());
    })
  )
);
```

### Vehicle Input and Dispatch Confirmation

```javascript
// From DispatchTransitCart.jsx - Vehicle number input handling
const [shipmentId, setShipmentId] = useState([null, ""]);

const handleShipmentIdChange = (event) => {
  let [error] = shipmentId;
  if (error && event.target.value) error = null;
  setShipmentId([error, event.target.value]);
};

const handleDispatch = async () => {
  let [error, value] = shipmentId;
  if (!value) error = "Please enter the Vehicle No.";
  if (error) return setShipmentId([error, value]);
  
  // Create dispatch events with vehicle information
  const response = await fetch(`${window.base_url}/app/shipment/event`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(
      shipments.map(({ request_code, shipment_code }) => ({
        request_code,
        shipment_code,
        shipment_vehicle: value,
      }))
    ),
  });
  
  clearCart();
  history.goBack();
};
```

## User Interface Patterns

### Panel Components Architecture

The application uses standardized panel components for consistent user experience:

#### DataPanel Pattern

Data presentation with integrated actions from the codebase:

```javascript
// From AcceptTealineWeight.jsx - DataPanel usage
<DataPanel
  title={selectedTealine.item_code}
  summary={[
    `Garden: ${selectedTealine.garden}`,
    `Broker: ${selectedTealine.broker}`,
    `Purchase Order: ${selectedTealine.purchase_order}`,
    `Created At: ${DateTime.fromMillis(
      Number(selectedTealine.created_ts)
    ).toFormat("hh:mm:ss a, dd/LL/yyyy")}`,
    `Bags Added: ${selectedTealine.record_list.length}`,
  ].join("\n")}
  enableText="Finish Item"
  onButtonClick={onFinishClick}
/>
```

#### SelectPanel Pattern

Selection interface with validation from the codebase:

```javascript
// From AcceptTealineSelect.jsx - SelectPanel usage
<SelectPanel
  title="Tealine - Accept Bags"
  subtitle="Select a tealine item to start accepting bags."
  select={{
    list: tealineList,
    key: (item) => item.item_code,
    item: (item) => `${item.item_code} - ${item.garden}`,
    hint: "Choose a tealine item...",
    errorHint: "Please select a tealine item.",
  }}
  panelState={resources}
  onStartClick={onSelectPanelStartClick}
/>
```

### Hardware Integration Patterns

#### Load Cell Integration

Real-time weight display from the codebase:

```javascript
// From AcceptTealineWeight.jsx - Load cell display
<Box
  mt={12}
  mb="auto"
  textAlign="center"
  color={locked ? "red.solid" : loadCellValue.stable ? "green.solid" : "gray.solid"}
>
  <Heading size="5xl">
    {Number.parseFloat(loadCellValue.reading).toFixed(2)}
  </Heading>
  <Heading mt={2} size="sm">
    KG
    {locked ? " (Locked)" : loadCellValue.stable ? " (Stable)" : ""}
  </Heading>
</Box>
```

#### Scanner Integration

Barcode scanning functionality from the codebase:

```javascript
// From ProcessBlendsheetScan.jsx - Scanner input handling
const handleBarcodeSubmit = async (event) => {
  event.preventDefault();
  const formData = new FormData(event.target);
  const barcode = formData.get("barcode");
  
  if (!barcode || barcode.trim() === "") {
    setError("Please enter a barcode");
    return;
  }
  
  // Process barcode scan
  await onBarcodeSubmit(barcode);
  event.target.reset();
};
```

## API Integration Patterns

### Central-Handler Communication

Standardized API communication patterns from the codebase:

```javascript
// From AcceptTealineSelect.jsx - API error handling pattern
fetch(`${window.base_url}/app/tealine`, {
  method: "GET",
  headers: { "Content-Type": "application/json" },
})
  .then(async (res) => {
    if (!res.ok)
      throw new Error(res.status === 500 ? "Internal Server Error" : await res.text());
    return res.headers.get("content-type").startsWith("application/json")
      ? await res.json()
      : await res.text();
  })
  .catch((e) => {
    // Error handling
  });
```

### AWS-Manager Integration via Proxy

Integration patterns through Central-Handler proxy:

```javascript
// From DispatchTransitSelect.jsx - AWS-Manager proxy communication
const transitList = await fetch(`${window.base_url}/app/shipment`);
// Central-Handler proxies to AWS-Manager for order data

// From DispatchTransitCart.jsx - Dispatch event creation via proxy
await fetch(`${window.base_url}/app/shipment/event`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify([{
    request_code,
    shipment_code,
    shipment_vehicle: value,
  }]),
});
// Central-Handler proxies to AWS-Manager for event creation
```

## Development Tools

### Serial Monitor System

Hardware communication monitoring from the codebase:

```javascript
// From SerialMonitor.jsx - Window management
useEffect(() => {
  if (!activeWindow.current) {
    activeWindow.current = window.open(
      "",
      "",
      ["resizable", "minimizable", "maximizable", "closable"]
        .map((property) => `${property}=false`)
        .concat(
          Object.entries({ title: "Serial Monitor", x: 32, y: 32, alwaysOnTop: true }).map(
            ([property, value]) => `${property}=${value}`
          )
        )
        .join()
    );
  }
});
```

## Navigation and Routing

### React Router Integration

Navigation patterns from the codebase:

```javascript
// From AcceptTealineSelect.jsx - Navigation with state transfer
const history = useHistory();

const onSelectPanelStartClick = ({ selectValue }) => {
  if (!selectValue) return;
  
  history.push("/tealine/weight", {
    item_code: selectValue.item_code,
    created_ts: Number(selectValue.created_ts),
  });
};

// From DispatchTransitSelect.jsx - Navigation with dispatch data
const onSelectPanelStartClick = ({ selectValue, shipQty }) => {
  history.push("/transit", { ...selectValue, ship_qty: Number.parseInt(shipQty) });
};
```

## Error Handling and Validation

### Resource State Management

Error handling patterns from the codebase:

```javascript
// From AcceptTealineSelect.jsx - Resource error handling
const [resources, updateResources] = useState(null);

try {
  // Resource setup logic
  updateResources({ status: "ready" });
} catch (e) {
  updateResources({
    status: "error",
    error: new SelectPanelError("Operation Failed", e.message),
  });
}
```

## Performance Optimization

### Memory Management

Cleanup patterns from the codebase:

```javascript
// From AcceptTealineWeight.jsx - Hardware resource cleanup
useEffect(() => {
  const listeners = [];
  
  // Setup listeners
  listeners.push(
    app.serialRx("loadcell", handleLoadCellData),
    app.serialRx("button", handleButtonPress)
  );
  
  // Cleanup
  return () => {
    listeners.forEach((cleanFn) => cleanFn());
  };
});
```

## Integration with MTP Platform

### Role-Based Workflow Integration

The Device-Panel workflows integrate with the role-based access control system:

**SEEDUWA Site Operations** (Device-Panel Focus):
- Hardware integration for material scanning, weighing, and printing
- Shipment allocation record creation (`*_allocate_shipment`)
- Order retrieval and dispatch event creation via Central-Handler proxy
- Physical dispatch operations and confirmation

**GRANDPASS Site Operations** (Minimal Context):
- Order management and status tracking
- Shipment receiving and acceptance verification
- Parcel allocation refinement via Admin-Panel (`*_allocate_parcel`)

### Traceability and Audit

The workflow implementation ensures complete traceability through allocation systems:

**Processing Traceability**:
- `tealine_allocate_blendsheet` + `blendbalance_allocate_blendsheet`: Ingredient tracking for blended products
- `blendsheet_allocate_flavorsheet` + `herbline_allocate_flavorsheet`: Base material and flavor ingredient tracking (planned)

**Dispatch Traceability**:
- Device-Panel: `*_allocate_shipment` creation for material dispatch
- Admin-Panel: `*_allocate_parcel` refinement for production scheduling
- Serial allocation ensures proper dependency management

**Cross-Site Status Progression**:
- **ACCEPTED**: Material received and validated at SEEDUWA
- **IN_PROCESS**: Material partially allocated to processes or shipments at SEEDUWA
- **PROCESSED**: Material fully allocated but not yet dispatched from SEEDUWA
- **DISPATCHED**: Material shipped from SEEDUWA to GRANDPASS
- **SHIPMENT_ACCEPTED**: Material received and verified at GRANDPASS

This comprehensive workflow implementation provides a robust foundation for tea processing operations at SEEDUWA, ensuring data integrity, complete traceability, and seamless integration with GRANDPASS operations through proper allocation-based material flow management and role-based access control.