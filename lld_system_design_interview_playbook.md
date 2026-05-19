# LLD & System Design Interview Playbook
*Companion to the Head First Design Patterns study guide. Walks through the most-asked design interview problems exactly the way you should solve them on a whiteboard or in a Zoom call.*

---

## How to attack ANY design interview (memorize this 6-step script)

Every problem — parking lot, payment gateway, Uber, doesn't matter — follows the same script. Interviewers are watching whether you have a method, not whether you know the "right answer".

| Step | Time | What to do | What to say out loud |
|---|---|---|---|
| 1. Clarify | 2-3 min | Functional + non-functional requirements, scale, scope. | "Before I jump in, can I confirm a few things? Are we building X for Y users? Should I include feature Z or focus on the core flows?" |
| 2. Define APIs / Use cases | 2 min | Top-level operations. For LLD: actors and use cases. For HLD: REST endpoints. | "So the system needs to support: book, cancel, pay, refund. Let me design the contracts first." |
| 3. Identify entities / components | 3 min | Pull nouns out of the requirements. Group them. | "The core entities I see are: User, Booking, Inventory, Payment..." |
| 4. Class diagram with patterns | 10 min | Relationships (has-a, is-a). Name patterns explicitly. | "I'll use a **Factory** here because we have multiple product variants, and a **Strategy** for pricing because pricing logic will change." |
| 5. Walk a happy-path flow | 5 min | Pick one realistic scenario. Trace method calls. | "Let me walk through a user booking a ticket end-to-end so we can see the objects collaborating." |
| 6. Edge cases / scale | 5 min | Concurrency, failure, evolution. | "If I had more time, here's how I'd handle concurrent bookings / payment failures / scaling to 10× the load." |

**Rules that score points:**
- **Talk before drawing.** Explain your thinking; don't go silent.
- **Name patterns explicitly.** "I'm using the Strategy pattern" beats "I'll make this swappable".
- **Justify with principles.** "I'm separating these because Single Responsibility" — interviewers love this.
- **Start small, expand.** MVP first. Don't try to design everything.
- **Admit trade-offs.** "I could use inheritance here, but composition gives me more flexibility at the cost of more objects."

---

# Problem 1: Parking Lot ⭐⭐⭐
*The #1 most-asked LLD interview question. If you can do this confidently, you can do most of the others.*

### Clarifying questions
- Single lot or multiple? **Single, multiple floors.**
- Vehicle types? **Motorcycle, Car, Truck.**
- Pricing model? **Hourly, with different rates per spot type.**
- Payments? **Mention it, but focus on park/unpark logic.**
- Multiple entry/exit gates? **Yes — important for concurrency.**

### Core entities
`ParkingLot`, `ParkingFloor`, `ParkingSpot` (abstract), `Vehicle` (abstract), `Ticket`, `EntryGate`, `ExitGate`, `FeeStrategy`, `DisplayBoard`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Singleton** | `ParkingLot` | Only one instance globally. |
| **Strategy** | `FeeStrategy` | Hourly / daily / flat-rate pricing swappable at runtime. |
| **Factory** | `VehicleFactory` (if needed) | Decouple vehicle creation. |
| **Observer** | `DisplayBoard` listens to spot availability | Multiple display boards update in real time. |
| **State** | `Ticket` (Active → Paid → Exited) | Models the ticket lifecycle cleanly. |

### Class sketch
```java
class ParkingLot {                          // Singleton
    private static ParkingLot instance;
    private List<ParkingFloor> floors;
    private FeeStrategy feeStrategy;
    private List<DisplayBoard> displays;    // Observers
    
    public synchronized Ticket parkVehicle(Vehicle v) {
        ParkingSpot spot = findSpot(v);
        if (spot == null) throw new LotFullException();
        spot.assign(v);
        notifyDisplays();
        return new Ticket(v, spot, Instant.now());
    }
    
    public double unparkVehicle(Ticket t) {
        t.markExit(Instant.now());
        double fee = feeStrategy.calculate(t);
        t.getSpot().release();
        notifyDisplays();
        return fee;
    }
}

abstract class ParkingSpot {
    protected String id;
    protected boolean occupied;
    protected Vehicle vehicle;
    public abstract boolean canFit(Vehicle v);
}

class CompactSpot extends ParkingSpot {
    public boolean canFit(Vehicle v) {
        return v.type() == CAR || v.type() == MOTORCYCLE;
    }
}

interface FeeStrategy {
    double calculate(Ticket t);
}

class HourlyFeeStrategy implements FeeStrategy { /* rate per hour by spot type */ }
class FlatRateStrategy implements FeeStrategy { /* fixed rate per day */ }
```

### Happy-path flow
1. Car arrives → `EntryGate.admit(car)` → `lot.parkVehicle(car)`.
2. Lot iterates floors, each floor returns first available compatible spot.
3. Spot is marked occupied, ticket issued, displays observe the change and decrement available count.
4. Exit → `ExitGate.process(ticket)` → `lot.unparkVehicle(ticket)` → fee strategy computes price.

### Follow-ups interviewers love
- **Concurrency at entry gates** → Pessimistic lock per floor, or use `ConcurrentLinkedQueue<ParkingSpot>` of free spots per type so reservation is atomic (`poll()`).
- **Lost ticket** → Add `LOST` state, charge a max-day rate.
- **Reserved spots** → Ticket has a "reserved-until" field; finder skips reserved spots that aren't expired.
- **Multi-lot company** → Lot is no longer a singleton — register lots with a `ParkingLotManager` (which is the singleton).

---

# Problem 2: Hotel Management / Booking System ⭐⭐⭐

### Clarifying questions
- One hotel or chain? **Chain — multiple properties.**
- Room types? **Single, Double, Suite, etc.**
- Booking horizon? **Future date ranges, not just check-in now.**
- Pricing? **Dynamic — varies by season, demand.**
- Cancellation? **Yes, with policies.**

### Core entities
`Hotel`, `Room` (abstract → `SingleRoom`, `DoubleRoom`, `Suite`), `RoomType`, `Booking`, `Guest`, `Payment`, `SearchService`, `PricingStrategy`, `NotificationService`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `PricingStrategy` | Seasonal, weekend, loyalty pricing — all swappable. |
| **Factory** | `RoomFactory` | Different room types instantiated based on input. |
| **State** | `Booking` (Pending → Confirmed → Checked-In → Checked-Out / Cancelled) | The booking lifecycle is a state machine. |
| **Observer** | Notifications | Email/SMS observers fire on state transitions. |
| **Singleton** | `HotelManagementSystem` | Global registry. |
| **Decorator** | Add-ons (breakfast, airport pickup) on a booking | Each add-on wraps the booking and adds to cost. |

### Class sketch
```java
class Hotel {
    private String id;
    private List<Room> rooms;
    
    public List<Room> searchAvailable(LocalDate in, LocalDate out, RoomType type) {
        return rooms.stream()
            .filter(r -> r.getType() == type)
            .filter(r -> r.isAvailable(in, out))
            .toList();
    }
}

class Room {
    private String number;
    private RoomType type;
    private List<DateRange> bookings;   // sorted, used for availability check
    
    public boolean isAvailable(LocalDate in, LocalDate out) {
        return bookings.stream().noneMatch(b -> b.overlaps(in, out));
    }
}

class Booking {                          // State pattern
    private BookingState state;          // PendingState, ConfirmedState, etc.
    private Guest guest;
    private Room room;
    private DateRange dates;
    private double totalCost;
    
    public void confirm() { state.confirm(this); }
    public void cancel()  { state.cancel(this); }
}

interface PricingStrategy { double price(Room r, DateRange dr); }
class SeasonalPricing implements PricingStrategy { /* peak/off-peak */ }
class DynamicPricing implements PricingStrategy { /* demand-based */ }
```

### Happy-path flow
1. Guest searches: hotel → list of available rooms.
2. Picks a room → `Booking` created in PENDING state.
3. Guest pays → `PaymentService.charge(...)` → on success, booking transitions to CONFIRMED.
4. Notification observers fire → email + SMS.
5. At check-in date → transition to CHECKED-IN.

### Follow-ups
- **Double booking under concurrency** → Optimistic locking on room availability, or use a unique constraint on `(room_id, date)` rows.
- **Overbooking strategy** → Allow N% overbook; if all show up, upgrade or comp.
- **Loyalty program** → Decorator on pricing, or a separate `DiscountStrategy`.
- **Multi-region scale** → Shard by hotel chain; each region has its own DB; eventual consistency for analytics.

---

# Problem 3: Movie / Event Ticketing System (BookMyShow / Ticketmaster) ⭐⭐⭐

### Clarifying questions
- City → cinema → screen → show → seat hierarchy? **Yes.**
- Seat selection or auto-assign? **User selects.**
- Hold time during selection? **Yes — typically 5-10 minutes.**
- Payment integrated or external gateway? **External gateway (so design the contract).**

### Core entities
`City`, `Cinema`, `Screen` (auditorium), `Show`, `Movie`, `Seat`, `Booking`, `Payment`, `User`, `SeatLock`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **State** | `Booking` (Created → SeatsHeld → Paid → Confirmed → Cancelled) | Clear lifecycle. |
| **Strategy** | `SeatPricingStrategy` (premium, standard), `PaymentStrategy` | Pluggable. |
| **Observer** | Notify users on booking confirmation, send refund alerts | Multiple channels. |
| **Singleton** | `SeatLockService` | Central lock manager. |
| **Factory** | `PaymentProcessorFactory` (Stripe / Razorpay / Paypal) | Hide concrete gateway. |

### The hardest part: the seat hold
This is what differentiates ticketing from hotel booking. When you click a seat, it must be **reserved for you for 5 minutes** while you pay, then released if you don't.

Two approaches — interviewers love hearing both:

**Approach A: In-memory lock with TTL (Redis).**
- `SETNX seat:{showId}:{seatId} userId EX 600` — atomic set-if-not-exists with 10-minute expiry.
- Pros: Fast, simple. Cons: Lost on Redis failure.

**Approach B: Database row with status + expiry.**
- `INSERT INTO seat_holds (...)` with unique constraint on `(show_id, seat_id, status='HELD')`.
- Background job cleans expired holds. Pros: Durable. Cons: Slower.

### Class sketch
```java
class Show {
    private Movie movie;
    private Screen screen;
    private LocalDateTime startTime;
    private Map<Seat, SeatStatus> seatStatus;  // AVAILABLE, HELD, BOOKED
}

class SeatLockService {                      // Singleton
    private RedisClient redis;
    
    public boolean tryLockSeats(String userId, String showId, List<String> seatIds) {
        // Lua script for atomic multi-seat lock — all-or-nothing
        // If any seat is already held/booked, fail and release everything just locked.
    }
    
    public void releaseSeats(...);
    public void confirmSeats(...);  // called after payment
}

class BookingService {
    public Booking createBooking(User u, Show s, List<Seat> seats) {
        if (!lockService.tryLockSeats(u.getId(), s.getId(), seatIds)) {
            throw new SeatsUnavailableException();
        }
        Booking b = new Booking(u, s, seats, Status.SEATS_HELD);
        // Booking now in SEATS_HELD state, payment pending
        return b;
    }
}
```

### Happy-path flow
1. User browses → city → cinema → show → seat map.
2. Selects 2 seats → `BookingService.createBooking()` → seats locked for 10 min, booking in `SEATS_HELD`.
3. User pays → `PaymentService.charge()` → on success, `lockService.confirmSeats()` → booking → `CONFIRMED`.
4. If user abandons → TTL expires → seats auto-released.
5. Observer fires → email ticket / QR code.

### Follow-ups
- **Sudden traffic spike (Tatkal / popular show)** → Queue + token system; only N users get the seat-selection page at once.
- **Same user opens 5 tabs** → Per-user limit on concurrent locks.
- **Refund flow** → Payment in REFUND_PENDING state; async reconciliation with payment gateway.

---

# Problem 4: Payment Gateway System ⭐⭐⭐

### Clarifying questions
- Internal payment service or fronting multiple processors (Stripe, Razorpay, Adyen)? **Fronting multiple — that's the interesting design.**
- Sync or async? **Both — card auth is sync, settlement is async.**
- Idempotency? **Yes, critical.**
- Refunds, chargebacks, recurring? **Yes — design extensibly.**

### Core entities
`PaymentRequest`, `PaymentResponse`, `Transaction`, `PaymentProcessor` (interface), `StripeProcessor` / `RazorpayProcessor` / etc., `PaymentRouter`, `IdempotencyStore`, `FraudCheckService`, `WebhookHandler`, `LedgerService`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `PaymentProcessor` | Each gateway is a strategy. Router picks one based on rules. |
| **Adapter** | Wrapping each external gateway's SDK in our `PaymentProcessor` interface | Each SDK has its own contract; we adapt. |
| **Factory** | `PaymentProcessorFactory` | Instantiate the right processor. |
| **Chain of Responsibility** | Pre-payment pipeline: fraud check → balance check → rate limit | Each handler decides to process/reject/forward. |
| **State** | `Transaction` (Initiated → Authorized → Captured → Settled / Refunded / Failed) | Full payment lifecycle. |
| **Observer** | Internal services react to settlements (ledger, accounting, user notification) | Decouple downstream consumers. |
| **Proxy** | Idempotency proxy in front of processor calls | Returns cached response for duplicate `idempotency_key`. |
| **Decorator** | Logging / metrics around processor calls | Cross-cutting concerns. |

### The two non-negotiables
**1. Idempotency.** Every payment request carries an `idempotency-key`. If client retries (network blip), we MUST return the *same* response, not double-charge.

```java
class IdempotencyProxy implements PaymentProcessor {
    private final PaymentProcessor real;
    private final IdempotencyStore store;
    
    public PaymentResponse charge(PaymentRequest r) {
        var cached = store.lookup(r.idempotencyKey());
        if (cached.isPresent()) return cached.get();
        
        var response = real.charge(r);
        store.save(r.idempotencyKey(), response, TTL_24H);
        return response;
    }
}
```

**2. The two-phase money flow.** Auth (reserve funds, no money moves) → Capture (money moves). Lets you ship-then-charge for marketplaces, and gives a window to cancel.

### Class sketch
```java
interface PaymentProcessor {
    PaymentResponse authorize(PaymentRequest r);
    PaymentResponse capture(String authId, Money amount);
    PaymentResponse refund(String transactionId, Money amount);
}

class StripeAdapter implements PaymentProcessor {     // Adapter
    private final StripeSDK sdk;
    public PaymentResponse authorize(PaymentRequest r) {
        var stripeCharge = sdk.charges.create(toStripeFormat(r));
        return fromStripeFormat(stripeCharge);
    }
}

class PaymentRouter {                                  // Strategy selection
    public PaymentProcessor pickProcessor(PaymentRequest r) {
        // Rules: currency, country, amount, gateway health, cost
        if (r.country() == "IN") return processors.get("razorpay");
        if (r.amount().value() > 10_000) return processors.get("adyen");
        return processors.get("stripe");
    }
}

// Pre-payment pipeline — Chain of Responsibility
abstract class PaymentHandler {
    protected PaymentHandler next;
    public abstract PaymentResponse handle(PaymentRequest r);
}
class FraudCheckHandler extends PaymentHandler { ... }
class RateLimitHandler  extends PaymentHandler { ... }
class IdempotencyHandler extends PaymentHandler { ... }
```

### Happy-path flow
1. Client `POST /pay` with `idempotency-key`.
2. Request enters chain: idempotency check → rate limit → fraud score.
3. Router picks `StripeAdapter` based on rules.
4. Adapter calls Stripe SDK → returns authorization.
5. Transaction state → AUTHORIZED. Ledger entry created (double-entry, in PENDING).
6. Async: capture worker triggers capture → state → CAPTURED → ledger settled.
7. Stripe later sends webhook for `payment_succeeded` → reconciler matches → state → SETTLED.

### Follow-ups
- **Eventual consistency across processor + ledger + accounting** → Outbox pattern: write a row in the same DB transaction as the payment record; a relay publishes to Kafka; consumers update downstream systems. Saga or 2PC are the buzzwords.
- **Gateway is down** → Circuit breaker (Hystrix-style) + fallback to secondary gateway.
- **Disputes/chargebacks** → New state + webhook handler that triggers a workflow.
- **PCI compliance** → Never store raw card numbers; tokenize via the gateway. Mention this even briefly — it earns points.

---

# Problem 5: ATM Machine ⭐⭐
*A short, focused LLD problem. The textbook State-pattern application.*

### Clarifying questions
- Operations? **Withdraw, deposit, balance, change PIN.**
- Card types? **Debit / credit.**
- Single bank or multi-bank? **Connect to bank backend via interface.**

### Core entities
`ATM`, `Card`, `Account`, `BankService` (interface), `CashDispenser`, `Screen`, `Keypad`, `Transaction`, `ATMState`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **State** | `ATM` (Idle → HasCard → Authenticated → SelectingOperation → Dispensing → Idle) | This is the canonical State pattern problem. |
| **Strategy** | `CashDispensingStrategy` (greedy / minimum-bills) | How to break down a withdrawal amount. |
| **Chain of Responsibility** | Withdraw pipeline: PIN check → balance check → daily limit → fraud check | Sequential validators. |
| **Facade** | `ATM` exposes a simple interface; hides cash dispenser, card reader, screen | Simplification. |

### Class sketch
```java
interface ATMState {
    void insertCard(Card c);
    void enterPin(String pin);
    void selectOperation(Operation op);
    void cancel();
}

class IdleState implements ATMState {
    public void insertCard(Card c) { atm.setCard(c); atm.setState(new HasCardState(atm)); }
    public void enterPin(String pin) { throw new IllegalStateException(); }
    // ...
}

class HasCardState implements ATMState {
    public void enterPin(String pin) {
        if (bank.verifyPin(atm.getCard(), pin)) atm.setState(new AuthenticatedState(atm));
        else atm.setState(new IdleState(atm));
    }
}
```

### Follow-ups
- **Cash dispenser runs out of a denomination** → Strategy returns "cannot dispense exact amount".
- **Network partition** → Offline mode with local cache + later reconciliation.
- **Concurrency** → ATM is single-user-at-a-time, but the *bank backend* has its own concurrency story.

---

# Problem 6: Vending Machine ⭐⭐
*Sister problem to the ATM. Same patterns.*

### Core entities
`VendingMachine`, `Inventory`, `Product`, `Coin/Note`, `VendingMachineState` (NoCoinInserted, CoinInserted, ProductSelected, DispensingProduct).

### Patterns
**State** (machine states) + **Strategy** (change-return strategies — greedy, dp) + **Observer** (low-stock alerts).

### The book uses a Gumball Machine for this — Chapter 10. Apply the same State pattern directly.

---

# Problem 7: Elevator System ⭐⭐⭐

### Clarifying questions
- Single elevator or bank? **Bank of N elevators.**
- Optimization goal? **Minimize average wait time.**
- Special modes? **VIP, maintenance, freight, fire emergency.**

### Core entities
`Elevator`, `ElevatorController` (the dispatcher), `Request` (`InternalRequest` from inside car, `ExternalRequest` from floor button), `Direction`, `ElevatorState`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `ElevatorSelectionStrategy` | Nearest car, look-ahead, even distribution — pluggable. |
| **State** | `Elevator` (Idle, MovingUp, MovingDown, Stopped, Maintenance) | Self-explanatory. |
| **Command** | `Request` | Each button press is a command queued in the controller. |
| **Observer** | Floor display, weight sensor, door sensor observe elevator state | Multiple watchers. |
| **Singleton** | `ElevatorController` | Central dispatcher for the bank. |

### The hard part: dispatching
```java
interface ElevatorSelectionStrategy {
    Elevator select(List<Elevator> elevators, ExternalRequest r);
}

class NearestCarStrategy implements ElevatorSelectionStrategy {
    public Elevator select(List<Elevator> evs, ExternalRequest r) {
        return evs.stream()
            .filter(e -> canServe(e, r))            // direction-aware
            .min(Comparator.comparingInt(e -> Math.abs(e.currentFloor() - r.floor())))
            .orElseThrow();
    }
}
```

`canServe` is the subtle bit: an elevator going up at floor 3 *can* serve an up-request at floor 5, but *can't* serve a down-request at floor 2 right now.

### Follow-ups
- **Star pattern (lobby-heavy)** → Idle elevators return to lobby.
- **Fire mode** → All elevators dispatch to ground floor, then disable.
- **Distributed control** → Each elevator has local logic + soft coordination via a controller; if controller dies, elevators still serve internal requests.

---

# Problem 8: Cab Booking System (Uber / Lyft / Ola) ⭐⭐⭐

### Clarifying questions
- LLD or HLD? **LLD focus, but mention geo-indexing.**
- Features? **Book, cancel, fare calc, driver matching, rating.**
- Real-time location? **Yes — drivers ping location every few seconds.**
- Surge pricing? **Yes.**

### Core entities
`Rider`, `Driver`, `Vehicle`, `Trip`, `Location`, `FareCalculator`, `MatchingService`, `PricingStrategy`, `RatingService`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `PricingStrategy` (normal, surge, scheduled) and `MatchingStrategy` (nearest, highest-rated) | Pluggable algorithms. |
| **State** | `Trip` (Requested → DriverAssigned → InProgress → Completed / Cancelled) | Trip lifecycle. |
| **Observer** | Notification on state transitions (rider, driver, ETA service) | Multi-channel. |
| **Singleton** | `LocationService` | Central registry. |
| **Factory** | `VehicleFactory` (Mini / Sedan / SUV / Auto) | Different products. |

### The geo problem
Finding the nearest driver to a rider is the interesting subproblem.

- **Naive:** For every driver, compute distance — O(N). Won't scale.
- **Better:** Spatial index — **Quadtree, R-tree, or geohash**. Mention by name. In production: Uber's H3 (hex grid), Google S2.
- **Practical sketch:** Bucket drivers by geohash prefix. Look up rider's geohash → query that bucket + 8 neighbors → distance-filter the small result set.

### Class sketch
```java
class Trip {                              // State pattern
    private TripState state;
    private Rider rider;
    private Driver driver;
    private Location pickup, dropoff;
    private FareCalculator fareCalc;
    
    public void start()    { state.start(this); }
    public void complete() { state.complete(this); }
}

interface PricingStrategy {
    Money calculate(Location from, Location to, VehicleType type, Instant when);
}

class SurgePricingStrategy implements PricingStrategy {
    public Money calculate(...) {
        double base = baseStrategy.calculate(...);
        double multiplier = demandIndex.getMultiplier(from, when);
        return base.multiply(multiplier);
    }
}

class MatchingService {
    public Optional<Driver> findDriver(Location pickup, VehicleType type) {
        List<Driver> candidates = geoIndex.nearby(pickup, RADIUS_KM);
        return candidates.stream()
            .filter(d -> d.getVehicle().type() == type)
            .filter(Driver::isAvailable)
            .min(Comparator.comparingDouble(d -> d.location().distanceTo(pickup)));
    }
}
```

### Follow-ups
- **Driver goes offline mid-trip** → Trip enters a "lost" state; retry, then offer rider a re-match.
- **Payment** → Hook into the payment gateway design from Problem 4.
- **Scheduled rides** → A separate `ScheduledTrip` entity that promotes to `Trip` at the booking time.

---

# Problem 9: Splitwise (Expense-Sharing App) ⭐⭐

### Clarifying questions
- Group expenses? **Yes.**
- Split types? **Equal, exact amounts, percentage, share-based.**
- Settle-up? **Yes — minimize transactions.**
- Currencies? **Multi-currency optional.**

### Core entities
`User`, `Group`, `Expense`, `Split` (abstract → `EqualSplit`, `ExactSplit`, `PercentSplit`), `Balance`, `Transaction`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `SplitStrategy` | Equal / exact / percent split logic. |
| **Factory** | `SplitFactory` | Create the right split type from input. |
| **Observer** | Notify members on new expense / settle-up | Email/push. |
| **Singleton** | `SplitwiseService` | Central balance manager. |

### The interesting algorithm: settle-up
After many expenses, the balance graph is dense. Naively you'd do N² transactions. Real Splitwise minimizes transactions by using a greedy algorithm:

1. For each user, compute net balance (positive = is owed, negative = owes).
2. Put creditors in a max-heap, debtors in a min-heap.
3. Pop top of each; settle the smaller amount; push the remainder back.
4. Repeat until both heaps are empty.

This produces O(N) transactions in the best case, O(N²) worst, but with a much smaller constant than the naive pairwise approach.

### Class sketch
```java
class Expense {
    private User paidBy;
    private Money amount;
    private List<Split> splits;
    
    public void apply(BalanceSheet bs) {
        for (Split s : splits) {
            bs.add(paidBy, s.user(), s.amount());  // paidBy is owed; s.user owes
        }
    }
}

abstract class Split {
    protected User user;
    protected Money amount;  // computed by strategy
}

class EqualSplit extends Split { /* amount = total / N */ }
class PercentSplit extends Split { /* amount = total * pct */ }
class ExactSplit  extends Split { /* amount given directly */ }

class BalanceSheet {
    // balances.get(A).get(B) = amount A owes B
    private Map<User, Map<User, Money>> balances;
    
    public List<Transaction> minimizeSettlements() {
        // Compute net, then greedy settle as above
    }
}
```

---

# Problem 10: Notification System ⭐⭐

### Clarifying questions
- Channels? **Email, SMS, push, in-app.**
- Priority? **Yes — transactional vs marketing.**
- Templating? **Yes.**
- Scale? **High — millions/day.**

### Core entities
`Notification`, `Recipient`, `Channel` (interface), `EmailChannel`, `SmsChannel`, `PushChannel`, `Template`, `NotificationQueue`, `RateLimiter`.

### Patterns used
| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `Channel` | Each channel is a strategy with its own send logic. |
| **Factory** | `ChannelFactory` | Pick channel by notification type. |
| **Decorator** | Rate limiting / retry / logging on send | Cross-cutting. |
| **Observer** | Delivery status callbacks (delivered, bounced, opened) | Multiple downstream consumers. |
| **Chain of Responsibility** | Pre-send pipeline: validate → check user preferences → throttle → templatize | Sequential. |
| **Builder** | `Notification.builder()...build()` | Many optional fields. |

### Class sketch
```java
interface Channel {
    DeliveryResult send(Notification n);
}

class EmailChannel implements Channel {
    public DeliveryResult send(Notification n) {
        var rendered = templateEngine.render(n.template(), n.data());
        return sesClient.send(n.recipient().email(), rendered);
    }
}

class RetryingChannel implements Channel {        // Decorator
    private final Channel inner;
    public DeliveryResult send(Notification n) {
        for (int i = 0; i < 3; i++) {
            var r = inner.send(n);
            if (r.success()) return r;
            sleep(backoff(i));
        }
        return DeliveryResult.failed();
    }
}

class NotificationDispatcher {
    public void dispatch(Notification n) {
        Channel ch = channelFactory.forType(n.type(), n.recipient());
        queue.enqueue(new SendTask(ch, n));
    }
}
```

### Follow-ups
- **Per-user channel preferences** → Each user has a preferences object; dispatcher consults it.
- **Quiet hours** → Schedule send for later.
- **Dedup** → Idempotency key on notifications; skip if seen recently.
- **High scale** → Kafka topic per priority; workers consume; backpressure via rate-limited channels.

---

# Problem 11: Library Management System ⭐

### Core entities
`Library`, `Book`, `BookItem` (physical copy — multiple per Book), `Member`, `Loan`, `Reservation`, `Fine`, `LibrarianService`.

### Patterns
- **State** — `BookItem` (Available, Loaned, Reserved, Lost, Damaged).
- **Strategy** — `FineCalculator` (per-day, escalating, capped).
- **Observer** — Member is notified when a reserved book becomes available.
- **Factory** — `BookFactory` (Book, Magazine, AudioBook).

### Class sketch
```java
class BookItem {
    private Book book;
    private BookItemState state;
    private Member currentBorrower;
    private LocalDate dueDate;
}

class Member {
    public Loan borrow(BookItem item) {
        if (loans.size() >= MAX_LOANS) throw new LimitException();
        item.setState(LOANED);
        return new Loan(this, item, LocalDate.now().plusDays(14));
    }
}
```

A nice "warm-up" problem before the harder ones. Doesn't usually go beyond 20 min of an interview.

---

# Problem 12: Chess / Tic-Tac-Toe ⭐⭐
*Tests pure OOP modeling. No infra/scale.*

### Core entities
`Game`, `Board`, `Cell`, `Piece` (abstract), `Pawn`/`Rook`/etc., `Player`, `Move`, `MoveValidator`.

### Patterns
- **Factory** — `PieceFactory` to create pieces on board setup.
- **Strategy** — Each `Piece` subclass implements its own `canMove(from, to, board)` — this is also classic polymorphism, not just Strategy.
- **Command** — Each `Move` is a command object (enables undo, replay, save-game).
- **Memento** — Save/restore board state.
- **Observer** — UI observes game state; AI engine observes moves.
- **State** — `Game` (NotStarted, InProgress, Check, Checkmate, Stalemate, Draw).

### Class sketch
```java
abstract class Piece {
    protected Color color;
    public abstract boolean isValidMove(Cell from, Cell to, Board b);
}

class Pawn extends Piece {
    public boolean isValidMove(Cell f, Cell t, Board b) {
        // forward one (or two from start), capture diagonal, en passant
    }
}

class Move {                          // Command
    private Cell from, to;
    private Piece captured;
    public void execute(Board b);
    public void undo(Board b);
}
```

For **Tic-Tac-Toe**, simplify: `Board` is 3×3, `Move` is just `(player, row, col)`, `Game` has a `checkWinner()` after each move. Interviewers want clean class boundaries, not algorithmic cleverness.

---

# Problem 13: Snake and Ladder ⭐

Single board, dice, players, snakes, ladders. Patterns:
- **Strategy** for dice (fair, loaded — useful for tests).
- **Factory** for board entities.
- **Observer** so UI updates on each move.

Walk through the move-execution loop: roll → move → check for snake/ladder → check for win.

---

# Problem 14: LRU Cache ⭐⭐
*A short LLD that doubles as a coding question.*

### Requirements
- `get(key)`: O(1).
- `put(key, value)`: O(1).
- Evict the least-recently-used when at capacity.

### The design
A `HashMap<K, Node>` + a **doubly-linked list** of `Node`s ordered by recency. The map gives O(1) lookup; the list gives O(1) move-to-front and O(1) tail-eviction.

```java
class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K,V>> map = new HashMap<>();
    private final Node<K,V> head, tail;     // sentinels
    
    public V get(K key) {
        Node<K,V> n = map.get(key);
        if (n == null) return null;
        moveToFront(n);
        return n.value;
    }
    
    public void put(K key, V value) {
        Node<K,V> n = map.get(key);
        if (n != null) { n.value = value; moveToFront(n); return; }
        if (map.size() == capacity) {
            Node<K,V> lru = tail.prev;
            removeNode(lru);
            map.remove(lru.key);
        }
        Node<K,V> fresh = new Node<>(key, value);
        addToFront(fresh);
        map.put(key, fresh);
    }
}
```

### Pattern angle
**Strategy** for the eviction policy — LRU today, LFU or FIFO tomorrow. Make `EvictionPolicy` an interface.

### Follow-ups
- **Thread-safe** → Synchronize methods, or use `ConcurrentHashMap` + striped locks. Or just wrap in `Collections.synchronizedMap` if perf doesn't matter.
- **Distributed LRU** → Hard. Mention consistent hashing for partitioning, and TTL-based eviction since coordinating recency across nodes is expensive.
- **Java has `LinkedHashMap` with access-order mode** → A one-liner LRU. Worth mentioning to show range.

---

# Problem 15: Rate Limiter ⭐⭐⭐

### Algorithms (know all four)
| Algorithm | Idea | Trade-offs |
|---|---|---|
| **Token Bucket** | Bucket has capacity C, refills at R tokens/sec. Each request consumes one token. | Allows bursts up to C. Most common. |
| **Leaky Bucket** | Queue of fixed size; requests drain at constant rate. | Smooths bursts. Adds latency. |
| **Fixed Window** | Count requests per N-second window. Reset at boundary. | Simple. Burst at window boundary (2× allowed in 1 sec). |
| **Sliding Window Log** | Store timestamps; count those in the last N sec. | Accurate. Memory-heavy. |
| **Sliding Window Counter** | Hybrid: weighted average of current and previous fixed windows. | Approximate. Cheap. Production winner. |

### Class sketch (Token Bucket)
```java
class TokenBucket {
    private final int capacity;
    private final double refillRate;        // tokens per second
    private double tokens;
    private Instant lastRefill;
    
    public synchronized boolean tryConsume() {
        refill();
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;
    }
    
    private void refill() {
        Instant now = Instant.now();
        double elapsed = Duration.between(lastRefill, now).toMillis() / 1000.0;
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefill = now;
    }
}

class RateLimiter {
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    public boolean allow(String userId) {
        return buckets.computeIfAbsent(userId, k -> new TokenBucket(100, 10)).tryConsume();
    }
}
```

### Pattern angle
**Strategy** for the algorithm choice. **Decorator** to compose limits (per-user + per-IP + per-endpoint, all applied in order).

### Distributed follow-up
Single-node bucket doesn't work across a cluster. Use Redis with atomic Lua scripts to update the bucket. Mention this — interviewers will ask.

---

# Problem 16: Logger ⭐
*Classic Chain of Responsibility example.*

`Logger` (abstract, holds `next`) → `InfoLogger` → `DebugLogger` → `ErrorLogger`. Each logger handles messages at or above its level and forwards. Mirror SLF4J/log4j.

Patterns: **Chain of Responsibility** (level filtering), **Singleton** (global logger), **Strategy** (output destination: file/console/network), **Decorator** (add timestamps, request IDs).

---

# Problem 17: Food Delivery System (Swiggy / Zomato / DoorDash) ⭐⭐⭐

### Core entities
`User`, `Restaurant`, `MenuItem`, `Cart`, `Order`, `DeliveryAgent`, `Address`, `Payment`, `Promo`.

### Patterns
- **State** — `Order` (Placed → Confirmed → Preparing → ReadyForPickup → OutForDelivery → Delivered / Cancelled).
- **Strategy** — `PricingStrategy` (taxes, surge, promos), `DeliveryAssignmentStrategy` (nearest agent / batched).
- **Observer** — Restaurant + agent + customer all subscribe to order state changes.
- **Decorator** — Promo codes wrap a cart and modify the total.
- **Factory** — `OrderFactory`.

### The interesting bit: agent assignment
Same geo-indexing problem as Uber. Plus **order batching** — group two nearby pickups for one agent on the same route. NP-hard variant of TSP; in practice use greedy heuristics.

### Follow-ups
- **Surge during dinner rush** → Dynamic pricing strategy + extra delivery fee.
- **Restaurant cancels mid-prep** → Auto-refund flow.
- **Live location tracking** → Agent app publishes location every 5 sec; customer's app subscribes via WebSocket / SSE.

---

# Problem 18: URL Shortener (TinyURL / bit.ly) ⭐⭐
*Lighter LLD, heavier HLD. Be ready for both angles.*

### LLD angle
- `URL`, `URLService`, `ShorteningStrategy` (base62 / hash / counter).
- **Strategy** for the shortening algorithm.
- **Singleton** for the ID generator.

### HLD angle (interviewers usually want this)
- **Encoding** — Base62 a monotonic counter (62^7 ≈ 3.5T URLs in 7 chars). Or hash the long URL and take first 7 chars (handle collisions).
- **Storage** — Read-heavy. Key-value store (DynamoDB, Cassandra) keyed on short ID.
- **Read path** — `GET /:short` → DB lookup → 301 redirect. Front with a cache (Redis), hit rate >90%.
- **Write path** — ID generator service (snowflake-like, or Redis `INCR`, or DB sequences sharded).
- **Custom aliases** — Try insert; on conflict, reject.
- **Analytics** — Fire-and-forget event to Kafka; downstream consumer counts.

---

# Problem 19: File System ⭐⭐
*Pure OOP. Composite pattern showcase.*

### Core entities
`Entry` (abstract) → `File`, `Directory`. `Directory` holds `List<Entry>`. This is the **Composite** pattern in its purest form.

```java
abstract class Entry {
    protected String name;
    protected Directory parent;
    public abstract long size();
    public abstract void print(String indent);
}

class File extends Entry {
    private byte[] content;
    public long size() { return content.length; }
}

class Directory extends Entry {
    private List<Entry> children = new ArrayList<>();
    public long size() { return children.stream().mapToLong(Entry::size).sum(); }
    public void add(Entry e) { e.parent = this; children.add(e); }
}
```

Add **Visitor** for operations (find, total-size-by-extension, search) without modifying `Entry`. Add **Iterator** for tree traversal. Mention **Memento** if asked about snapshots.

---

# Problem 20: Data Pipeline (Data Engineering interview) ⭐⭐⭐

This is a different beast — it's an architecture question, not pure OOP. The interviewer wants to see you reason about volume, latency, schemas, and failure.

### Clarifying questions (asks them all)
- **Source(s)?** Database CDC, application logs, API events, IoT sensors, third-party files?
- **Sink(s)?** Data warehouse (Snowflake/BigQuery/Redshift), data lake (S3 + Parquet), real-time dashboard, ML feature store?
- **Latency budget?** Daily batch is one design; sub-second streaming is another.
- **Volume?** GBs/day or TBs/hour matters for tech choice.
- **Schema stability?** Stable contract or evolving?
- **Reprocessing?** Can we replay historical data?

### Reference architecture: Lambda → Kappa
Most pipelines fall into one of three shapes.

**Batch (cheapest, simplest):**
```
Sources → (S3 / GCS landing zone) → Airflow / dbt → Warehouse
```
- Use when latency >1 hour is acceptable.
- Cron / Airflow DAGs orchestrate. dbt for SQL transformations.

**Streaming:**
```
Sources → Kafka → Flink / Spark Streaming → Sink
```
- Use for sub-minute latency: fraud detection, real-time dashboards.
- Kafka is the backbone — durable, replayable, partitioned.

**Lambda (batch + streaming side by side):**
```
                    ┌─→ Batch (Spark) ──→ Warehouse (corrected layer)
Kafka ─→ (raw) ─────┤
                    └─→ Stream (Flink) ──→ Warehouse (speed layer)
```
- Two pipelines, results merged at query time.
- Acknowledge: this is operationally painful (two codebases). Modern teams prefer **Kappa** — streaming only, with replay for "batch" semantics.

### The four layers every interviewer wants to hear
1. **Ingestion** — pull or push. Kafka Connect for DBs (CDC via Debezium), Kinesis for AWS-native, simple HTTP receivers for APIs.
2. **Storage** — raw zone (S3 + Parquet, immutable), staging zone (cleaned), curated zone (modeled). Often called **medallion / bronze-silver-gold**.
3. **Processing** — Spark for batch, Flink for streaming, dbt for SQL transformations in-warehouse.
4. **Serving** — warehouse for analytics, feature store for ML, materialized views for dashboards.

### Patterns and principles applied
- **Idempotency** — every step must be safely re-runnable. Use deterministic keys, upsert (`MERGE`), not append.
- **Schema-on-write vs schema-on-read** — lake (read), warehouse (write). Use both: lake for raw + cheap storage, warehouse for fast analytics.
- **Schema registry** — Avro / Protobuf + Confluent Schema Registry. Avoids silent breakage when producers evolve schemas.
- **Exactly-once vs at-least-once** — exactly-once in Kafka + Flink is real but operationally heavy. Default to at-least-once + idempotent sinks; only push for exactly-once if duplicates are unacceptable.
- **Backfilling** — design for it from day 1. Replays from Kafka or re-runs over historical S3 partitions.
- **Data quality** — Great Expectations or dbt tests as a layer; nullability, uniqueness, freshness, row count thresholds.
- **Observability** — lineage (OpenLineage), freshness (Monte Carlo, Datafold), volume anomaly detection.

### A worked example: "Design a pipeline for an e-commerce site's order events"
1. **Source** — orders DB. Use Debezium CDC → Kafka topic `orders.cdc`.
2. **Raw landing** — Kafka Connect S3 sink writes Parquet files to `s3://lake/raw/orders/dt=YYYY-MM-DD/`.
3. **Streaming view** — Flink job consumes `orders.cdc`, computes 5-minute rolling revenue by category, writes to a serving DB for the real-time dashboard.
4. **Batch warehouse load** — dbt models build the curated `fact_orders` table in Snowflake daily, incremental on `updated_at`.
5. **Quality gates** — dbt tests on row counts, uniqueness of `order_id`, no nulls in `customer_id`.
6. **Lineage / alerting** — failures page the data team via PagerDuty.

### Follow-ups
- **GDPR / right-to-be-forgotten** in an append-only lake → Hold PII in a separately encrypted column with per-user keys; drop the key on deletion request.
- **Late-arriving data** → Watermarks in Flink; allowed lateness threshold.
- **Schema evolution** → Backward-compatible changes only (add nullable columns); use Avro/Protobuf forward compatibility rules.
- **Cost** → Tier storage (hot in warehouse, cold on S3). Partition + cluster well.

---

# Quick reference: pattern → problem cheat sheet

When the interviewer says "design X", these are the patterns to reach for first.

| If the problem involves... | Reach for... |
|---|---|
| A lifecycle with discrete stages (order, booking, ticket) | **State** |
| Pluggable algorithms (pricing, matching, sorting) | **Strategy** |
| Wrapping external SDKs into a unified interface | **Adapter** |
| Simplifying a multi-system operation | **Facade** |
| Controlling access (auth, lazy load, remote) | **Proxy** |
| Notification / event broadcast | **Observer** (in-process) or pub-sub (distributed) |
| Encapsulating an operation as a thing (undo, queue, log) | **Command** |
| One global instance (config, lot, factory) | **Singleton** (but consider DI first) |
| Creating one of several variants | **Factory Method** or **Abstract Factory** |
| Many optional construction params | **Builder** |
| Tree-shaped data (filesystem, menus, UI) | **Composite** |
| Adding cross-cutting behavior dynamically | **Decorator** |
| Sequential pipeline of validators/handlers | **Chain of Responsibility** |
| Algorithm skeleton with replaceable steps | **Template Method** |
| Traversing a collection uniformly | **Iterator** |
| Coordinating many peer objects | **Mediator** |
| Saving/restoring object state | **Memento** |

---

# Practice protocol

Pick 5-7 problems from this list and **draw each one out by hand** on paper or a whiteboard. Then *speak the design out loud* — record yourself if alone — for 20 minutes per problem. Two hours of this is worth ten hours of passive re-reading.

Order I'd practice for a 1-week prep:
1. **Day 1:** Parking Lot, ATM, Vending Machine (warm up; State pattern muscle memory).
2. **Day 2:** Hotel Booking, Movie Ticketing (lifecycle + concurrency).
3. **Day 3:** Payment Gateway, Notification System (cross-cutting concerns, adapters, chains).
4. **Day 4:** Uber, Splitwise (geo / algorithm angle).
5. **Day 5:** Elevator, Library, File System (pure OO).
6. **Day 6:** LRU Cache, Rate Limiter, URL Shortener (LLD-meets-HLD).
7. **Day 7:** Data Pipeline + review the pattern cheat sheet + your weakest 3.

Good luck out there.
