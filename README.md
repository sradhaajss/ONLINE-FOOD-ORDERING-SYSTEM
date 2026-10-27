# Online Food Ordering System

A complete, runnable full-stack food ordering platform with smart order management,
rule-based personalised recommendations, automatic delivery assignment and a Help Centre.

**Stack:** React 18 + Vite + Tailwind CSS + React Router · Java 17 + Spring Boot 3.2 (Spring Web, Spring Data JPA, Hibernate, Spring Security) · MySQL 8

```
React (Vite)  →  REST API  →  Service layer  →  Spring Data JPA / Hibernate  →  MySQL
```

---

## 1. What this project does

Four roles share one system, each seeing only what their role allows:

| Role | What they do |
|---|---|
| **Customer** | Browse restaurants, see live availability, add to cart, apply coupons, checkout with a simulated payment, track the order on a timeline, cancel within the allowed window, review a delivered order, use the Help Centre |
| **Restaurant** | Accept → prepare → mark ready, work through the Smart Order Queue, override a suggested priority, manage the menu, manage stock |
| **Delivery agent** | Receive assigned orders, mark out for delivery, mark delivered, go online/offline |
| **Admin** | Platform-wide dashboards, users, restaurants, agents, orders, FAQs |

---

## 2. Setup

### Prerequisites

| Tool | Version |
|---|---|
| JDK | 17 or newer |
| Maven | 3.8+ (or use the IDE's bundled Maven) |
| Node.js | 18 or newer |
| MySQL | 8.0 |

### 2.1 Database

The backend creates the database and tables by itself on first run
(`createDatabaseIfNotExist=true` and `spring.jpa.hibernate.ddl-auto=update`), and
`DataSeeder.java` fills it with demo data. So the quickest path is: **make sure MySQL is
running, then start the backend.**

If you'd rather create everything manually (useful for a viva or for the project report):

```bash
mysql -u root -p < database/schema.sql
mysql -u root -p food_ordering < database/sample_data.sql
```

If you load `sample_data.sql` manually, set `app.seed.enabled=false` in
`backend/src/main/resources/application.properties` so the Java seeder doesn't insert the
same rows again.

### 2.2 Backend (port 8080)

```bash
cd backend
mvn spring-boot:run
```

First run prints `>> Seeding demo data...` and finishes with the demo credentials.
Check it's alive: <http://localhost:8080/api/health>

### 2.3 Frontend (port 5173)

```bash
cd frontend
npm install
npm run dev
```

Open <http://localhost:5173>. Vite proxies every `/api` call to `http://localhost:8080`,
so there's nothing else to configure.

Production build: `npm run build` (output in `frontend/dist`).

---

## 3. Demo credentials

Every account uses the password **`pass123`**.

| Role | Email | Notes |
|---|---|---|
| Customer | `customer@tiffinroute.com` | Sri Harini — has order history, so recommendations are populated |
| Customer | `karthik@tiffinroute.com` | Has live orders in the queue |
| Customer | `meera@tiffinroute.com` | |
| Customer | `vignesh@tiffinroute.com` | |
| Restaurant | `spice@tiffinroute.com` | **Spice Route Kitchen** — the restaurant used in the demo script |
| Restaurant | `coastal@tiffinroute.com` | Coastal Catch |
| Restaurant | `garden@tiffinroute.com` | Green Garden Veg |
| Restaurant | `wok@tiffinroute.com` | Wok & Roll |
| Delivery agent | `suresh@tiffinroute.com` | Available |
| Delivery agent | `arun@tiffinroute.com` | Currently assigned |
| Delivery agent | `fathima@tiffinroute.com` | Currently out for delivery |
| Delivery agent | `rahul@tiffinroute.com` | Available |
| Admin | `admin@tiffinroute.com` | Platform dashboards |

The login page has one-click buttons for the four main demo accounts.

---

## 4. Configuration values you may need to change

All of these live in `backend/src/main/resources/application.properties`:

| Property | Default | Change it when |
|---|---|---|
| `spring.datasource.username` | `root` | Your MySQL user isn't root |
| `spring.datasource.password` | `root` | **Almost always** — set your own MySQL password |
| `spring.datasource.url` | `jdbc:mysql://localhost:3306/food_ordering?...` | MySQL runs on another host/port, or you want a different database name |
| `server.port` | `8080` | Port 8080 is already taken (also update `vite.config.js` proxy target) |
| `app.seed.enabled` | `true` | Set to `false` if you loaded `sample_data.sql` manually, or to stop re-seeding |
| `app.cors.allowed-origins` | `http://localhost:5173,...` | You run the frontend on a different port |
| `spring.jpa.hibernate.ddl-auto` | `update` | `create-drop` to rebuild the schema on every run |

In `frontend/vite.config.js`, `server.proxy['/api'].target` must match the backend port.

One more, in `backend/.../service/TokenService.java`: `SECRET` is a hard-coded demo key.
It's fine for a college project, but it's the first thing you'd move to an environment
variable if this went further.

---

## 5. Features implemented

### Core
- Registration and login for four roles, BCrypt-hashed passwords, HMAC-signed tokens, role-based route guards on both the API (`@PreAuthorize`) and the UI (`ProtectedRoute`)
- Restaurant and menu browsing with search, category filters and a veg-only filter
- Cart (one restaurant per order), coupon codes with minimum-order rules, delivery fee free above ₹499
- Checkout with simulated UPI / Card / COD payments and a `payments` record per order
- Order timeline, order history, restaurant reviews on delivered orders
- Admin dashboards for users, restaurants, agents, orders and FAQs

### The seven differentiating features

**1. Personalised recommendations ("Recommended For You")** — `RecommendationService.personalised()`.
Pure Java rules over the customer's own order history: +40 for a category they order often,
+25 for a keyword match (chicken, paneer, biryani…), +15 for a familiar restaurant, +10 for a
repeat dish, plus a popularity tie-breaker. A new customer with no history sees popular dishes
instead. Each card shows *why* it was recommended. No ML anywhere.

**2. "Customers Also Ordered"** — `ItemPairCount` + a `HashMap` index in `RecommendationService`.
Every placed order increments a count for each ordered pair of dishes. The food details page
asks for the top partners of that dish; the cart asks for partners of everything in it. Falls
back to popular dishes from the same restaurant when a dish has no pair history yet.

**3. Real-time stock-linked availability** — `FoodItem.refreshAvailability()`.
`stock > threshold` → AVAILABLE · `0 < stock ≤ threshold` → LOW_STOCK, shown as "Only 3 left" ·
`stock = 0` → OUT_OF_STOCK, greyed out with Add to cart disabled. Stock drops on every order,
is re-checked at checkout (so a race for the last portion is caught), and is restored when an
order is cancelled. Restaurants edit stock and the threshold on the Stock page.

**4. Smart order prioritisation** — `PriorityService`, using `PriorityQueue` with a `Comparator`.
Score out of 100 = waiting time (≤40) + urgency against the promised time (≤30) + longest prep
time (≤20) + item count (≤10), banded into HIGH / MEDIUM / LOW. The Smart Order Queue page ranks
live orders, explains any order's score on demand, and lets staff **override** a priority or
clear the override. It's decision support: the kitchen decides, the system suggests.

**5. Delivery agent assignment** — `DeliveryService.assignAgent()`.
When a restaurant marks an order READY, the next AVAILABLE agent in a round-robin rotation is
assigned and the order moves to ASSIGNED. Agent states run AVAILABLE → ASSIGNED →
OUT_FOR_DELIVERY → AVAILABLE. If everyone is busy, `NoAgentAvailableException` is thrown, the
order stays READY, and the restaurant can retry assignment later.

**6. Help Centre** — `HelpCentreService`, 7 categories (My Order, Payment, Cancellation & Refund,
Food Availability, Delivery, Account & Login, Other Issues), ~22 articles, keyword search across
questions, answers and tags, expandable accordion, a contact option, and a **context-aware panel**
that reads the customer's live order or a specific dish and explains the current situation in
plain words.

**7. Rule-based cancellation** — `OrderService.cancelOrder()`.
Allowed only while the order is PLACED or CONFIRMED. Anything later throws
`InvalidCancellationException`, which carries `helpTopic = "cancellation-refund"`; the React
toast turns that into a direct link to the right Help Centre section. A successful cancellation
restores stock and marks any prepaid payment REFUNDED.

### Order lifecycle

```
PLACED → CONFIRMED → PREPARING → READY → ASSIGNED → OUT_FOR_DELIVERY → DELIVERED
   └────── CANCELLED (only from PLACED or CONFIRMED)
```
The restaurant owns the first three transitions; the delivery agent owns the last two.
Every other transition is rejected by the service layer.

---

## 6. Project structure

```
online-food-ordering-system/
├── README.md
├── .gitignore
├── frontend/
│   ├── package.json  vite.config.js  tailwind.config.js  postcss.config.js  index.html
│   └── src/
│       ├── main.jsx  App.jsx  index.css
│       ├── components/   Navbar, Sidebar, FoodCard, RestaurantCard, CartItem, OrderCard,
│       │                 AvailabilityBadge, RecommendationCard, FAQAccordion, StatusBadge,
│       │                 PriorityBadge, OrderTimeline, StarRating, ProtectedRoute,
│       │                 LoadingState, EmptyState, ErrorState, Modal
│       ├── pages/
│       │   ├── auth/        Login, Register
│       │   ├── customer/    Home, Restaurants, RestaurantMenu, FoodDetails, Cart, Checkout,
│       │   │                MyOrders, OrderTracking, HelpCentre, Profile
│       │   ├── restaurant/  RestaurantLayout, RestaurantDashboard, RestaurantOrders,
│       │   │                SmartQueue, MenuManagement, StockManagement
│       │   ├── delivery/    DeliveryDashboard
│       │   └── admin/       AdminDashboard
│       ├── services/    api.js, endpoints.js, useApi.js
│       └── context/     AuthContext, CartContext, ToastContext
├── backend/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/foodordering/
│       │   ├── FoodOrderingApplication.java
│       │   ├── model/       User, Restaurant, Category, FoodItem, CartItem, Order, OrderItem,
│       │   │                DeliveryAgent, ItemPairCount, Payment, Coupon, Review, Faq, FaqCategory
│       │   ├── repository/  14 Spring Data JPA repositories
│       │   ├── service/     Auth, Catalog, Cart, Order, Recommendation, Priority, Delivery,
│       │   │                HelpCentre, Review, Dashboard, Token, DtoMapper
│       │   ├── controller/  Auth, Catalog, Cart, Order, Recommendation, RestaurantOps,
│       │   │                Delivery, HelpCentre, Review, Coupon, Admin
│       │   ├── dto/         Request/response shapes
│       │   ├── exception/   Custom exceptions + GlobalExceptionHandler
│       │   ├── enums/       Role, OrderStatus, Availability, AgentStatus, PaymentMethod,
│       │   │                PaymentStatus, PriorityLevel
│       │   └── config/      SecurityConfig, TokenAuthFilter, DataSeeder
│       └── resources/application.properties
├── database/
│   ├── schema.sql
│   └── sample_data.sql
└── docs/
    └── diagrams.md
```

---

## 7. Database tables

`users` · `restaurants` · `categories` · `food_items` · `cart_items` · `orders` · `order_items` ·
`item_pair_counts` · `delivery_agents` · `payments` · `coupons` · `reviews` · `faqs` · `faq_categories`

---

## 8. Demo script

A run-through that shows every feature in about five minutes.

1. Sign in as `customer@tiffinroute.com` / `pass123`.
2. The home page shows **Recommended for you**, built from this customer's chicken-heavy history.
3. Open **Spice Route Kitchen**. Ennai Kathirikai is greyed out (sold out); Mutton Dum Biryani shows "Only 4 left".
4. Open **Chicken Dum Biryani** → **Customers also ordered** suggests Chicken 65, Onion Raita and Cola with the number of times they were ordered together.
5. Add the biryani, open the cart, apply **TRICHY20**, go to checkout, pick UPI, place the order.
6. Go back to the dish: its stock count has dropped.
7. Sign in as `spice@tiffinroute.com`. The new order appears on the dashboard and in **Smart Order Queue** with a priority score.
8. Click **Why this position?** to see the score breakdown, then **Override priority** to move it by hand.
9. Accept → Start cooking → Mark ready. A delivery agent is assigned automatically at READY.
10. Sign in as that agent (e.g. `suresh@tiffinroute.com`): the order is waiting. Mark it out for delivery, then delivered.
11. Back as the customer: the timeline is complete. Submit a review.

**Cancellation rule:** while an order is PLACED or CONFIRMED, "Cancel order" works and stock is
restored. Once the restaurant moves it to PREPARING, cancelling is refused and the message links
straight to *Cancellation & Refund* in the Help Centre.

---

## 9. API reference (short version)

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth/register`, `/api/auth/login` | Accounts |
| GET | `/api/restaurants`, `/api/restaurants/{id}/menu` | Browsing |
| GET | `/api/foods`, `/api/foods/{id}`, `/api/foods/popular` | Dishes |
| GET/POST/PUT/DELETE | `/api/cart/**` | Cart |
| POST | `/api/orders` | Place an order |
| PUT | `/api/orders/{id}/status`, `/api/orders/{id}/cancel` | Lifecycle |
| GET | `/api/recommendations/{customerId}` | Recommended for you |
| GET | `/api/recommendations/frequently-bought-together/{foodId}` | Customers also ordered |
| GET | `/api/restaurant/{id}/queue` | Smart Order Queue |
| PUT | `/api/restaurant/orders/{id}/priority` | Manual priority override |
| PUT | `/api/restaurant/menu/{foodId}/stock` | Stock update |
| GET/PUT | `/api/delivery/**` | Agent assignments and status |
| GET | `/api/faqs`, `/api/faqs/support` | Help Centre |
| GET | `/api/admin/**` | Admin dashboards |

---

## 10. Troubleshooting

| Problem | Fix |
|---|---|
| `Cannot reach the server` in the UI | The backend isn't running on 8080, or the Vite proxy target is wrong |
| `Access denied for user 'root'@'localhost'` | Set the right `spring.datasource.username` / `password` |
| `Unknown database 'food_ordering'` | Keep `createDatabaseIfNotExist=true` in the JDBC URL, or run `schema.sql` |
| Seed data doesn't appear | It only seeds when the `users` table is empty. Drop the database and restart, or run `sample_data.sql` |
| Duplicate demo rows | You ran both `sample_data.sql` and the Java seeder. Set `app.seed.enabled=false` |
| Port 8080 in use | Change `server.port` and the proxy target in `vite.config.js` |

---

Built as an academic project. Payments are simulated end to end — no gateway is contacted and
no card details are stored anywhere.
