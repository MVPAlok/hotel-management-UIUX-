# Booking System - Component Architecture & State Flow Guide

## 📐 Architecture Overview

This document provides a deep dive into the component structure, state management patterns, and data flow of the hotel booking system.

---

## 🏗️ Component Hierarchy

```
App (Root)
└── BookingProvider (Context Wrapper)
    └── BookingFlow
        ├── BookingHeader
        │   └── Navigation & Logo
        ├── StepIndicator (Steps 1-5)
        │   ├── Step badges
        │   └── Progress connector lines
        └── Dynamic Step Component (rendered based on state.currentStep)
            ├── 1. StayDetailsStep
            ├── 2. RoomSelectionStep
            ├── 3. GuestDetailsStep
            ├── 4. PaymentStep
            └── 5. ConfirmationStep
```

---

## 🔄 State Flow Architecture

### Initial State

```javascript
{
  currentStep: 1,
  stayDetails: { checkIn: '', checkOut: '', adults: 1, children: 0 },
  selectedRoom: null,
  guestDetails: { firstName: '', lastName: '', email: '', phone: '', specialRequests: '' },
  paymentDetails: { cardNumber: '', expiry: '', cvc: '' },
  confirmation: null,
  isLoading: false,
  error: null
}
```

### State Update Flow

```
Component → handleChange() → dispatch(action) → Reducer → New State → Component Re-render
```

### Example: Setting Check-in Date

```javascript
// User types date in input
<input onChange={(e) => handleChange('checkIn', e.target.value)} />

// Handler dispatches action
const handleChange = (field, value) => {
  dispatch({ type: 'SET_STAY_DETAILS', payload: { [field]: value } });
}

// Reducer processes action
case 'SET_STAY_DETAILS':
  return {
    ...state,
    stayDetails: { ...state.stayDetails, ...action.payload }
  };

// Component reads updated state
const { state } = useBooking();
const { stayDetails } = state;
```

---

## 🎯 Detailed Component Specifications

### 1. StayDetailsStep Component

**Responsibilities:**

- Collect dates and guest count
- Validate date ranges
- Calculate number of nights
- Display info box with stay summary

**Props:** None (uses context)

**State Management:**

- Dispatches `SET_STAY_DETAILS` for each field
- Dispatches `SET_CURRENT_STEP` to advance
- Local error state for validation messages

**Validation Logic:**

```javascript
function validateStayDetails(stayDetails) {
  const errors = [];

  // Rule 1: Both dates required
  if (!stayDetails.checkIn || !stayDetails.checkOut) {
    errors.push("Both dates required");
  }

  // Rule 2: Check-out after check-in
  if (new Date(stayDetails.checkOut) <= new Date(stayDetails.checkIn)) {
    errors.push("Check-out must be after check-in");
  }

  // Rule 3: No past dates
  const today = new Date().toISOString().split("T")[0];
  if (stayDetails.checkIn < today) {
    errors.push("Check-in cannot be in past");
  }

  // Rule 4: At least 1 guest
  const totalGuests =
    parseInt(stayDetails.adults) + parseInt(stayDetails.children);
  if (totalGuests === 0) {
    errors.push("At least 1 guest required");
  }

  return errors;
}
```

**Key Calculations:**

```javascript
// Nights calculation
const nights = Math.ceil(
  (new Date(checkOut) - new Date(checkIn)) / (1000 * 60 * 60 * 24)
);

// Total guests
const totalGuests = parseInt(adults) + parseInt(children);
```

**Transitions:**

- ✓ Valid → Step 2 (Room Selection)
- ✗ Invalid → Display error, stay on Step 1

---

### 2. RoomSelectionStep Component

**Responsibilities:**

- Filter rooms by capacity
- Display room options
- Handle room selection
- Calculate and display pricing
- Show amenity icons

**Props:** None (uses context)

**State Management:**

- Reads `stayDetails` for night calculation
- Reads `selectedRoom` for visual feedback
- Dispatches `SET_SELECTED_ROOM`
- Dispatches `SET_CURRENT_STEP`

**Room Filtering Logic:**

```javascript
// Filter rooms that can accommodate guest count
const totalGuests =
  parseInt(stayDetails.adults) + parseInt(stayDetails.children);
const availableRooms = roomsDatabase.filter(
  (room) => room.capacity >= totalGuests
);
```

**Price Calculation:**

```javascript
const nights = Math.ceil(
  (new Date(stayDetails.checkOut) - new Date(stayDetails.checkIn)) /
    (1000 * 60 * 60 * 24)
);
const totalPrice = selectedRoom.price * nights;
```

**Room Card Features:**

- Image with hover zoom effect
- Amenity icons (clickable for tooltip)
- Price per night and total
- Selection highlight (border & shadow)
- Capacity display

**Amenity Icon Map:**

```javascript
const amenityIcons = {
  wifi: "wifi",
  tv: "tv",
  bathtub: "bathtub",
  ac: "ac_unit",
  desk: "desk",
  kitchen: "kitchen",
  lock: "lock",
  spa: "spa",
  pool: "pool",
};
```

**Transitions:**

- Back → Step 1
- ✓ Room selected → Step 3
- ✗ No room selected → Show error

---

### 3. GuestDetailsStep Component

**Responsibilities:**

- Collect guest contact information
- Validate form fields
- Handle special requests
- Display form errors

**Props:** None (uses context)

**State Management:**

- Reads/writes `guestDetails` via context
- Local state for form errors

**Field Validation Rules:**

```javascript
const fieldValidation = {
  firstName: (value) =>
    value.trim().length > 0 ? null : "First name required",
  lastName: (value) => (value.trim().length > 0 ? null : "Last name required"),
  email: (value) => (value.includes("@") ? null : "Valid email required"),
  phone: (value) => (value.trim().length > 0 ? null : "Phone required"),
  specialRequests: () => null, // Optional
};
```

**Form Structure:**

```
[First Name]  [Last Name]
[Email Address]
[Phone Number]
[Special Requests - Textarea]
```

**Key Features:**

- Real-time error clearing on input
- Focus states with border color change
- Placeholder text for guidance
- Optional special requests field

**Transitions:**

- Back → Step 2
- ✓ Valid form → Step 4
- ✗ Invalid form → Show errors, stay on Step 3

---

### 4. PaymentStep Component

**Responsibilities:**

- Collect payment card details
- Format and validate card input
- Display security indicators
- Show live price calculation
- Handle payment processing
- Display order summary

**Props:** None (uses context)

**State Management:**

- Reads `paymentDetails` from context
- Dispatches `SET_PAYMENT` for field updates
- Dispatches `SET_LOADING` during processing
- Dispatches `SET_CONFIRMATION` on success
- Dispatches `SET_CURRENT_STEP` for navigation

**Card Input Formatting:**

```javascript
// Card Number - Format as 1234 5678 9012 3456
function formatCardNumber(value) {
  const cleaned = value.replace(/\s/g, "");
  return cleaned
    .slice(0, 16)
    .replace(/(\d{4})/g, "$1 ")
    .trim();
}

// Expiry - Format as MM/YY
function formatExpiry(value) {
  const cleaned = value.replace(/\D/g, "");
  if (cleaned.length >= 2) {
    return cleaned.slice(0, 2) + "/" + cleaned.slice(2, 4);
  }
  return cleaned;
}

// CVC - Only digits, max 4
function formatCVC(value) {
  return value.replace(/\D/g, "").slice(0, 4);
}
```

**Validation Rules:**

```javascript
function validatePayment(paymentDetails) {
  const errors = [];

  // Card number: 16 digits
  const cardClean = paymentDetails.cardNumber.replace(/\s/g, "");
  if (!cardClean.match(/^\d{16}$/)) {
    errors.push("Card number must be 16 digits");
  }

  // Expiry: MM/YY format
  if (!paymentDetails.expiry.match(/^\d{2}\/\d{2}$/)) {
    errors.push("Expiry must be MM/YY");
  }

  // CVC: 3-4 digits
  if (!paymentDetails.cvc.match(/^\d{3,4}$/)) {
    errors.push("CVC must be 3-4 digits");
  }

  return errors;
}
```

**Price Breakdown Calculation:**

```javascript
const nights = Math.ceil(
  (new Date(checkOut) - new Date(checkIn)) / (1000 * 60 * 60 * 24)
);
const subtotal = selectedRoom.price * nights;
const tax = subtotal * 0.1; // 10% tax
const total = subtotal + tax;

// Display format:
// Room × 3 nights: $1050.00
// Tax (10%):       $105.00
// Total:           $1155.00
```

**Payment Processing Flow:**

```javascript
async function handlePayment() {
  // 1. Validate card details
  const errors = validatePayment(paymentDetails);
  if (errors.length > 0) {
    setError(errors[0]);
    return;
  }

  // 2. Start loading
  dispatch({ type: "SET_LOADING", payload: true });

  // 3. Process payment (simulated 2 seconds)
  try {
    // In production: Send to payment gateway
    // await stripe.confirmCardPayment(paymentIntent);

    await new Promise((resolve) => setTimeout(resolve, 2000));

    // 4. Create confirmation data
    const confirmation = {
      confirmationNumber: "LUX-" + Date.now().toString().slice(-8),
      bookingId: generateRandomId(),
      // ... other data
    };

    // 5. Update state
    dispatch({ type: "SET_CONFIRMATION", payload: confirmation });
    dispatch({ type: "SET_CURRENT_STEP", payload: 5 });
  } catch (error) {
    setError(error.message);
  } finally {
    dispatch({ type: "SET_LOADING", payload: false });
  }
}
```

**Layout:**

- Left: Payment form (2/3 width on desktop)
- Right: Order summary (1/3 width, sticky on desktop)

**Transitions:**

- Back → Step 3
- ✓ Payment success → Step 5 (Confirmation)
- ✗ Payment error → Show error, stay on Step 4

---

### 5. ConfirmationStep Component

**Responsibilities:**

- Display success state
- Show confirmation details
- Provide next steps guide
- Allow email resend
- Provide navigation options

**Props:** None (uses context, reads confirmation)

**State Management:**

- Reads `confirmation` from context
- No state mutations (read-only component)

**Confirmation Data Structure:**

```javascript
{
  confirmationNumber: 'LUX-1734567890', // LUX- + 8-digit timestamp
  bookingId: 'ABC123DEF', // 9-char random
  guestName: 'John Doe',
  email: 'john@example.com',
  roomName: 'Deluxe Ocean View',
  checkIn: '2024-12-20',
  checkOut: '2024-12-25',
  nights: 5,
  total: 1750.00,
  date: '2024-12-15 14:30:45'
}
```

**Display Sections:**

1. **Success Icon** - Animated green checkmark
2. **Confirmation Details**
   - Confirmation number (copy-able)
   - Booking ID
   - Guest name
   - Room name
   - Check-in/out dates
   - Total amount
3. **Next Steps**
   - Email sent to guest
   - Mobile app check-in (24h before)
   - Concierge contact (48h before)
   - Enjoy your stay!
4. **Action Buttons**
   - Back to Home
   - Resend Confirmation Email

**Key Features:**

- Success animation (pulsing green icon)
- Clear confirmation numbers for reference
- Helpful next steps guidance
- Professional confirmation design

**Transitions:**

- Home → Back to index.html
- Email → Simulated send (alert in demo)

---

## 🔌 Data Integration Points

### 1. Room Data Integration

**Current (Mock Data):**

```javascript
const roomsDatabase = [
  { id: 1, name: '...', price: 350, ... },
  // ...
];
```

**For Production:**

```javascript
// In RoomSelectionStep:
useEffect(() => {
  fetchAvailableRooms(stayDetails.checkIn, stayDetails.checkOut).then(
    (rooms) => {
      setRooms(rooms);
    }
  );
}, [stayDetails]);
```

### 2. Payment Integration

**Current (Simulated):**

```javascript
await new Promise((resolve) => setTimeout(resolve, 2000));
```

**For Production (Stripe Example):**

```javascript
const response = await stripe.confirmCardPayment(clientSecret, {
  payment_method: {
    card: cardElement,
    billing_details: { name: guestDetails.firstName },
  },
});
```

### 3. Booking Creation Integration

**For Production:**

```javascript
// After payment succeeds:
const bookingResponse = await fetch("/api/bookings", {
  method: "POST",
  body: JSON.stringify({
    stayDetails,
    selectedRoom,
    guestDetails,
    paymentToken: paymentId,
    total: calculatedTotal,
  }),
});

const confirmation = await bookingResponse.json();
dispatch({ type: "SET_CONFIRMATION", payload: confirmation });
```

### 4. Email Integration

**For Production:**

```javascript
// Send confirmation email
await fetch("/api/emails/confirmation", {
  method: "POST",
  body: JSON.stringify({
    to: guestDetails.email,
    bookingId: confirmation.bookingId,
    guestName: guestDetails.firstName,
    // ... other data
  }),
});
```

---

## 🎨 Styling Architecture

### TailwindCSS Custom Classes

**Colors:**

```css
.primary {
  color: #5be830;
}
.bg-primary {
  background-color: #5be830;
}
.bg-background-dark {
  background-color: #152111;
}
.bg-surface-dark {
  background-color: #1e261c;
}
.bg-surface-dark-lighter {
  background-color: #2a3627;
}
```

**Custom Animations:**

```css
.fade-in {
  animation: fadeIn 0.3s ease-in;
}
.slide-in-left {
  animation: slideInLeft 0.4s ease-out;
}
.pulse-green {
  animation: pulse-green 2s infinite;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

**Reusable Patterns:**

```html
<!-- Input Pattern -->
<input
  class="bg-background-dark border border-white/10 rounded-lg px-4 py-3 text-white focus:border-primary focus:outline-none transition-colors"
/>

<!-- Button Pattern -->
<button
  class="rounded-full bg-primary hover:bg-primary/90 text-background-dark font-bold transition-all shadow-lg shadow-primary/20"
/>

<!-- Card Pattern -->
<div class="bg-surface-dark border border-white/5 rounded-2xl p-8" />
```

---

## 🧩 Reusable Logic Functions

### Night Calculator

```javascript
function calculateNights(checkIn, checkOut) {
  return Math.ceil(
    (new Date(checkOut) - new Date(checkIn)) / (1000 * 60 * 60 * 24)
  );
}
```

### Price Calculator

```javascript
function calculatePricing(roomPrice, nights, taxRate = 0.1) {
  const subtotal = roomPrice * nights;
  const tax = subtotal * taxRate;
  const total = subtotal + tax;
  return { subtotal, tax, total };
}
```

### Date Formatter

```javascript
function formatDate(dateString) {
  return new Date(dateString).toLocaleDateString("en-US", {
    year: "numeric",
    month: "long",
    day: "numeric",
  });
}
```

### Confirmation Number Generator

```javascript
function generateConfirmationNumber() {
  return "LUX-" + Date.now().toString().slice(-8);
}
```

---

## 📊 State Diagram

```
START
  ↓
┌─────────────────────────────────────┐
│  STEP 1: STAY DETAILS               │
│  Input: dates, guests               │
│  Validation: date range, guest count│
└─────────────────────────────────────┘
  ↓ (valid)
┌─────────────────────────────────────┐
│  STEP 2: ROOM SELECTION             │
│  Input: select room                 │
│  Validation: capacity match         │
└─────────────────────────────────────┘
  ↓ (room selected)
┌─────────────────────────────────────┐
│  STEP 3: GUEST DETAILS              │
│  Input: name, email, phone          │
│  Validation: email, required fields │
└─────────────────────────────────────┘
  ↓ (valid form)
┌─────────────────────────────────────┐
│  STEP 4: PAYMENT                    │
│  Input: card number, expiry, CVC    │
│  Validation: card format            │
└─────────────────────────────────────┘
  ↓ (payment success)
┌─────────────────────────────────────┐
│  STEP 5: CONFIRMATION               │
│  Display: booking summary           │
│  Actions: home, resend email        │
└─────────────────────────────────────┘
  ↓
END

(Users can go back at any step)
```

---

## 🔍 Debugging Tips

### Enable Console Logging

```javascript
// Add to reducer:
const bookingReducer = (state, action) => {
  console.log("Action:", action.type, action.payload);
  // ... reducer logic
  console.log("New State:", newState);
  return newState;
};
```

### Check Context Value

```javascript
const { state, dispatch } = useBooking();
console.log("Current booking state:", state);
```

### Monitor Step Changes

```javascript
useEffect(() => {
  console.log("Moved to step:", state.currentStep);
}, [state.currentStep]);
```

### Validate Form Data

```javascript
const handleNext = () => {
  console.log("Form validation:", {
    firstName: guestDetails.firstName,
    email: guestDetails.email,
    // ... other fields
  });
};
```

---

## 📋 Testing Scenarios

### Scenario 1: Happy Path

```
1. Select dates (12/20 to 12/25)
2. Select 2 adults, 0 children
3. Choose "Deluxe Ocean View" room
4. Enter guest details
5. Enter valid card (any 16 digits)
6. Verify confirmation displayed
```

### Scenario 2: Validation Testing

```
1. Try to go to Step 2 without selecting dates → Error
2. Try checkout before checkin → Error
3. Try Step 3 with invalid email → Error
4. Try Step 4 with invalid card → Error
```

### Scenario 3: Navigation Testing

```
1. Go Step 1 → 2 → 3 → back to 2 → 3 → 4
2. Verify data preserved at each step
3. Verify no data loss when navigating back
```

---

## 🚀 Performance Considerations

### Optimization Strategies

1. **Memoization**

   ```javascript
   const StepIndicator = React.memo(({ currentStep }) => {...});
   ```

2. **Lazy Loading**

   ```javascript
   const RoomCard = React.lazy(() => import("./RoomCard"));
   ```

3. **Input Debouncing** (for backend calls)

   ```javascript
   const debouncedSearch = useCallback(
     debounce((value) => fetchRooms(value), 300),
     []
   );
   ```

4. **Conditional Rendering**
   ```javascript
   {
     isLoading && <LoadingSpinner />;
   }
   {
     !isLoading && <PaymentForm />;
   }
   ```

---

This architecture provides a scalable, maintainable foundation for a production-grade hotel booking system.
