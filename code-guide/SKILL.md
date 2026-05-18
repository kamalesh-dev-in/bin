---
name: code-guide
description: Review and refactor any Express backend to follow the YLT coding structure. Enforces 4-layer architecture — routes (wiring only), controllers (io-prefix methods), HighLevel (business logic, extends LowLevel), LowLevel (raw operations). Checks MyResponse usage, LogService tags match filenames, HighLevel extends LowLevel with this.method() calls. Use when the user says "check structure", "refactor routes", "clean up backend", "review controller", "add endpoint", "create route", "create service", "refactor service", or when editing files under any backend's routes, controllers, or services directories.
allowed-tools: ["Read", "Glob", "Grep", "Bash"]
---

# YLT Backend Structure

Universal coding structure for Express.js backends. Works across any project — adapts to the route format used (Router vs app.route) while enforcing the same controller/service/response patterns.

Reference implementation: `node_speedbird_app` (`src/app/routes/`, `src/app/functions/`, `src/app/service/`)

---

## Architecture

```
Request → Route (wiring only) → Middleware → Controller (io function) → HighLevel (business logic, extends LowLevel) → LowLevel (raw operations) → Model
```

Four layers, strict separation:
- **Routes** — pure wiring, zero logic
- **Controllers** — `io` prefix routing functions, parse request, call high level, return response
- **HighLevel** — business logic, orchestration, `extends LowLevel`, calls `this.method()`
- **LowLevel** — raw operations, single DB/API calls, data fetching

---

## 1. Routes

Routes are wiring only. No business logic, no inline handlers (except trivial redirects).

### Route formats

The route definition style varies by project — both are valid:

**Format A — `express.Router()`** (flat, one line per route):
```typescript
import { Router } from "express";
import AdminController from "../admin/admin_controller";
import UserController from "../user/user_controller";

const router = Router();

router.post("/api/admin/users", AdminController.ioListMergedUsers);
router.post("/api/admin/users/block", UserController.ioBlockUser);
router.get("/api/agents/:id/history", AgentController.ioAgentHistory);

export default router;
```

**Format B — `app.route().method()`** (chaining pattern):
```typescript
import VehicleController from "../functions/vehicle_controller";
import auth from "../functions/google/auth/auth_controller";

let authenticated = '/auth';
let authenticatedAdmin = '/auth/admin';

export default function (app) {
    app.route(authenticated + '/get_vehicle_list')
        .post(auth.ioVerifyToken2, VehicleController.ioGetVehicleList);

    app.route(authenticatedAdmin + '/user_activation')
        .post(auth.ioVerifyToken2, auth.ioVerifyAdminDealer, VehicleController.ioUserActivationByAdmin);
}
```

Both formats follow the same rules. Detect which format the project uses and stay consistent.

### Rules

- **Zero logic** — routes are pure wiring, one line per endpoint
- **No inline handlers** — every handler is a controller method (except trivial redirects)
- **No comments** — the route definition is self-documenting
- **Group by controller** — keep routes for the same controller together
- **Middleware as positional args** — auth first, then role checks, then handler
- **POST for authenticated endpoints** — even data retrieval
- **GET for public/static endpoints** only

### Middleware patterns

Middleware may be applied per-route or at mount level — both valid:

**Per-route (Format B):**
```typescript
app.route(authenticated + '/get_vehicle_list')
    .post(auth.ioVerifyToken2, VehicleController.ioGetVehicleList);
```

**At mount level (Format A):**
```typescript
// index.ts
app.use("/auth", authMiddleware, requireAccess, authRoutes);
app.use("/auth", authMiddleware, requireAccess, requireAdmin, adminRoutes);
```

---

## 2. Controllers

**Controllers manage the routing functions (io functions).**

They parse the request, call services, and return `MyResponse`. No business logic.

### Pattern

```typescript
import MyResponse from "./response_controller";
import LogService from "../service/log_service";
import { VehicleModel } from "../mongo_models/vehicle_model";
import MyUtil from "../util/my_util";

const logger = new LogService('vehicle_controller');

export default class VehicleController {

    static async ioGetVehicleList(req, res) {
        let userId = req.body.decodedUser?.user?.userId;

        if (MyUtil.isNullOrUndefined(userId))
            return res.json(MyResponse.getFailureResponse("unauthorized"));

        let vehicles = await VehicleModel.dbGetVehiclesForUser(userId);

        let result = MyResponse.getSuccessResponse();
        result.data = vehicles;
        res.json(result);
    }

    static async ioBlockUser(req, res) {
        try {
            let userId = req.body.userId;
            if (MyUtil.isNullOrUndefined(userId))
                return res.json(MyResponse.getFailureResponse("missing_user_id"));

            let result = await UserLowLevel.blockUser(userId);
            res.json(MyResponse.getSuccessResponse("Blocked", result));
        } catch (e) {
            res.json(MyResponse.getFailureResponse(`Error in ioBlockUser : ${e}`));
        }
    }
}
```

### Rules

- **`io` prefix** on all handler methods — e.g., `ioGetVehicleList`, `ioBlockUser`
- **Static methods** — no instances needed for most controllers
- **Early returns** for error cases — no deep nesting (max 2 levels)
- **try/catch** wrapping each handler — catch and return failure response
- **Delegate to services** — controllers are thin, no business logic
- **Logger tag = snake_case file name** — `new LogService('vehicle_controller')`
- **`export default`** class — PascalCase class name matching file name

### Handler naming convention

```
io + verb + object

ioGetVehicleList       // get = direct fetch
ioCreateNewVehicle     // create = builds new object
ioDeleteVehicle        // delete
ioBlockUser            // block/unblock = state change
ioGrantUserRole        // grant/revoke = permission change
ioEmailVehicleReport   // email/send = notification
ioAgentStart           // start/stop = lifecycle
```

### Where user ID comes from

Depends on the auth middleware — check the project's pattern:

| Pattern | Source | Example |
|---|---|---|
| Firebase cookie auth | `res.locals.userId` | agent47 |
| Firebase token auth | `req.body.decodedUser?.user?.userId` | speedbird_app |

---

## 3. Services

**Services process the data. Business logic resides here.**

No `req`/`res`. HTTP-agnostic. Organized as LowLevel + HighLevel pairs.

### LowLevel — "How" (raw operations)

Single operations, direct DB/API calls, data fetching. No business logic, no orchestration. Returns raw data.

```typescript
// auth_low_level.ts
import UserLowLevel from "../user/user_low_level";

class AuthLowLevel {
  static async getUserById(userId: string) {
    return await UserLowLevel.getUserById(userId);
  }
}

export default AuthLowLevel;
```

### HighLevel — "What" (business logic) — `extends LowLevel`

Business rules, validation, orchestration. **Must extend the LowLevel class.** Calls LowLevel methods via `this.`.

```typescript
// auth_high_level.ts
import { UserRole } from "../mongo_model/user_model";
import MyUtil from "../services/my_util";
import AuthLowLevel from "./auth_low_level";
import LogService from "../services/log_service";

const logger = new LogService("auth_high_level");

class AuthHighLevel extends AuthLowLevel {
  static async getUserProfile(userId: string) {
    const user = await this.getUserById(userId);  // inherited from LowLevel
    if (MyUtil.isNullOrUndefined(user)) return null;

    const roleArray = user!.roleArray ?? [];
    const hasAccessRole = roleArray.includes(UserRole.saUser) || roleArray.includes(UserRole.admin);
    const isAllowed = user!.isActive !== false && hasAccessRole;
    const isAdmin = roleArray.includes(UserRole.admin);

    return {
      userId: user!.userId,
      userEmail: user!.currentEmail,
      displayName: user!.displayName,
      roleArray,
      isAllowed,
      isAdmin,
    };
  }
}

export default AuthHighLevel;
```

### Directory structure

Each domain gets its own folder with three files:

```
backend/src/<domain>/
├── <domain>_low_level.ts      # Raw operations (DB, API, file system)
├── <domain>_high_level.ts     # Business logic (extends LowLevel)
└── <domain>_controller.ts     # HTTP layer (calls HighLevel)
```

### Full flow example

```
Route:   router.get("/api/me", AuthController.ioMe)
Controller: AuthController.ioMe → AuthHighLevel.getUserProfile(userId)
HighLevel:  AuthHighLevel.getUserProfile → this.getUserById(userId) (inherited)
LowLevel:   AuthLowLevel.getUserById → UserLowLevel.getUserById → Model
```

### Rules

- **No `req`/`res`** — services are HTTP-agnostic
- **HighLevel `extends` LowLevel** — inheritance, not composition
- **Use `this.method()`** — not `ClassName.method()` to call inherited LowLevel methods
- **Return raw values** — booleans, objects, arrays, null for not-found
- **LowLevel** — single operations, no orchestration, no business rules
- **HighLevel** — orchestration, validation, multi-step flows, cron jobs
- **File name = snake_case** — e.g., `vehicle_low_level.ts`, `vehicle_high_level.ts`
- **Class name = PascalCase** — e.g., `VehicleLowLevel`, `VehicleHighLevel`
- **`export default`** class on both LowLevel and HighLevel
- **Interfaces** exported from LowLevel (e.g., `IScopeBound`, `IAdminMergedUser`)
- **Delegate to MongoDB models** — LowLevel calls `Model.dbXXX()` methods

---

## 4. Common Utilities

### MyResponse

Standardized response format used across controllers and services.

```typescript
import MyResponse from "./response_controller";

// Success
let result = MyResponse.getSuccessResponse();
result.data = someData;
res.json(result);

// Success with message and data
res.json(MyResponse.getSuccessResponse("Success", { userArray }));

// Failure
return res.json(MyResponse.getFailureResponse("Error message"));

// Failure with data
res.json(MyResponse.getFailureResponse("error", { details: "..." }));

// Checking
if (MyResponse.isFailure(result)) return res.json(result);
```

Interface:
```typescript
interface IBaseResponse {
    response?: number;   // 200 = success, -1 = failure, 429 = rate limited
    message?: string;
    data?: any;
}
```

### LogService

Instance-based logger. Tag must match file name (snake_case).

```typescript
import LogService from "../service/log_service";

const logger = new LogService('vehicle_controller');
logger.log({ message: 'something happened', data: value });     // always prints
logger.debug({ step: 'processing', count: 5 });                  // debug mode only
logger.err({ error: 'something broke' });                        // error

// With options:
const saveLogger = new LogService('tag', {
    isCountEnabled: true,    // adds counter to each log line
    isSaveLog: true,         // saves to MongoDB (production only)
    saveChannel: 'custom'    // custom log channel
});
```

---

## 5. Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Route files | `snake_case_routes.ts` | `auth_routes.ts`, `production_routes.ts` |
| Controller files | `snake_case_controller.ts` | `vehicle_controller.ts` |
| Service files | `snake_case_low_level.ts` + `snake_case_high_level.ts` | `vehicle_low_level.ts`, `vehicle_high_level.ts` |
| Class names | `PascalCase` | `VehicleController`, `VehicleService` |
| LogService tags | `snake_case` (match file name) | `new LogService('vehicle_controller')` |
| Handler methods | `io` + PascalCase | `ioGetVehicleList`, `ioBlockUser` |
| Service methods | `camelCase` | `isDeviceImeiModificationAllowed` |
| Model methods | `db` + PascalCase | `dbGetVehicleById`, `dbFindAndSetVehicleById` |
| Middleware methods | `io` + PascalCase | `ioVerifyToken2`, `ioVerifyAdminDealer` |
| Private methods | `_` prefix | `_ioGetTollsInternal` |
| Route paths | `snake_case` | `/get_vehicle_list`, `/grant-role` |

---

## 6. Review Checklist

When reviewing or refactoring any backend, check:

### Routes
- [ ] Routes contain zero business logic
- [ ] No inline handlers (except trivial redirects)
- [ ] Every handler is a controller method reference
- [ ] Middleware is consistent (per-route or mount-level, not mixed)
- [ ] Routes grouped by controller

### Controllers
- [ ] All handler methods have `io` prefix
- [ ] Static methods (no unnecessary instances)
- [ ] Early returns for error cases (no deep nesting)
- [ ] try/catch wrapping each handler
- [ ] No business logic — delegates to HighLevel
- [ ] Uses `MyResponse` for all responses
- [ ] Logger tag matches file name

### HighLevel
- [ ] `extends` corresponding LowLevel class
- [ ] Calls LowLevel methods via `this.`, not `ClassName.`
- [ ] Business logic, orchestration, validation
- [ ] No `req`/`res` parameters
- [ ] `export default` class

### LowLevel
- [ ] Single operations, raw DB/API calls
- [ ] No business logic, no orchestration
- [ ] No `req`/`res` parameters
- [ ] Interfaces exported from here
- [ ] `export default` class

### General
- [ ] File names are snake_case
- [ ] Class names are PascalCase
- [ ] LogService tag matches file name
- [ ] No unnecessary comments
- [ ] No hardcoded values that belong in config

---

## 7. Refactoring Steps

When cleaning up routes that have inline logic:

1. **Identify inline handlers** — grep for `(req, res) =>` or `async (req, res)` in route files
2. **Create `<domain>/` directory** — new folder for the domain's service layer
3. **Create `<domain>_low_level.ts`** — extract raw operations (DB calls, API calls, file reads)
4. **Create `<domain>_high_level.ts`** — extract business logic, `extends LowLevel`, use `this.method()`
5. **Create `<domain>_controller.ts`** — add `io` prefix static methods, call HighLevel, return `MyResponse`
6. **Replace inline with controller reference** — route becomes one-liner
7. **Place route in correct file** — based on auth level (public / auth / admin)
8. **Remove old file** if all logic moved to the new structure

### Example refactoring

**Before** — inline handler in route file:
```typescript
// routes/auth_routes.ts
router.get("/api/me", async (req, res) => {
  const user = await UserLowLevel.getUserById(res.locals.userId);
  // ... 30 lines of role checking, response building ...
});
```

**After** — proper layered structure:
```typescript
// auth/auth_low_level.ts
class AuthLowLevel {
  static async getUserById(userId: string) {
    return await UserLowLevel.getUserById(userId);
  }
}
export default AuthLowLevel;

// auth/auth_high_level.ts
class AuthHighLevel extends AuthLowLevel {
  static async getUserProfile(userId: string) {
    const user = await this.getUserById(userId);
    // ... role checking ...
  }
}
export default AuthHighLevel;

// auth/auth_controller.ts
class AuthController {
  static async ioMe(_req: Request, res: Response): Promise<void> {
    const profile = await AuthHighLevel.getUserProfile(res.locals.userId);
    res.json(MyResponse.getSuccessResponse("Me", profile));
  }
}
export default AuthController;

// routes/auth_routes.ts
router.get("/api/me", AuthController.ioMe);
```
