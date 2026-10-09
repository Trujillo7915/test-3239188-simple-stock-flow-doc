# Technical Security Rules — Simple Stock Flow

> **Mandatory code-level security controls for C#, ASP.NET Core, EF Core, and PostgreSQL.**

---

## 1. OWASP Top 10 Controls for Simple Stock Flow

### A01: Broken Access Control
- Every API endpoint (except `/api/v1/auth/login`) MUST require authentication.
- Role checks (`admin`, `seller`) MUST be validated in the Application Layer / Use Case, not solely in UI controllers.
- Report queries (`Q9`) and product write operations (`HU-CAT-02`, `HU-CAT-03`) MUST enforce `Role == "admin"`.

### A02: Cryptographic Failures
- Passwords MUST be hashed using `IPasswordHasher` using BCrypt (work factor >= 12) or Argon2id (`D-09`).
- Sensitive configurations (DB connection strings, JWT secret keys, storage credentials) MUST be loaded from environment variables (`Environment.GetEnvironmentVariable`), NEVER hardcoded.
- Application logs MUST scrub headers and bodies containing credentials.

### A03: Injection (SQL & Command Injection)
```csharp
// ❌ VULNERABLE: Direct SQL concatenation
var query = $"SELECT * FROM sales.product WHERE name = '{userInput}'";

// ✅ SECURE: Entity Framework Core LINQ (fully parameterized)
var product = await dbContext.Products
    .Where(p => p.Name == userInput && p.DeletedAt == null)
    .FirstOrDefaultAsync();

// ✅ SECURE: Parameterized raw SQL for optimized reports (D-06, Q9)
var report = await dbContext.Database
    .SqlQueryRaw<ReportItemDto>(
        "SELECT product_id, product_name, category_name, SUM(quantity) as total_qty " +
        "FROM sales.sale_item si JOIN sales.sale s ON s.id = si.sale_id " +
        "WHERE s.sold_at >= {0} AND s.sold_at <= {1} " +
        "GROUP BY product_id, product_name, category_name", startDate, endDate)
    .ToListAsync();
```

### A04: Insecure Design & Concurrency Collsions
- Stock deduction MUST use optimistic locking via PostgreSQL `xmin` (`D-04`, `T-10`).
- Database check constraint `ck_product_stock_non_negative` MUST exist as the ultimate safety net (`ADR-002`).

### A05: Security Misconfiguration
- Production environments MUST disable developer exception pages (`UseDeveloperExceptionPage`).
- Standard HTTP security headers enabled:
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `Content-Security-Policy: default-src 'self'`
- Database user connecting from API MUST only possess privileges on schema `sales` (`GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA sales`).

### A06: Identification & Authentication Failures
- Rate limiting on `/api/v1/auth/login`: Maximum 5 failed attempts per IP per 5 minutes.
- Username normalization: `username.Trim().ToLowerInvariant()` executed before lookup to prevent spoofing and duplicate collisions (`§2.5`).
- JWT tokens signed with asymmetric key or 256+ bit symmetric key, with maximum 1-hour expiration.

---

## 2. Input Validation at Adapter Layer

All external HTTP requests must pass through validation schemas (e.g., FluentValidation) before instantiating domain models:

```csharp
public class RegisterSaleRequestValidator : AbstractValidator<RegisterSaleRequest>
{
    public RegisterSaleRequestValidator()
    {
        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("A sale must contain at least one line item (Sale.EnsureConfirmable §2.3)");
            
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId).NotEmpty();
            item.RuleFor(i => i.Quantity).GreaterThan(0).WithMessage("Quantity must be strictly positive (§2.4)");
        });
    }
}
```

---

## 3. Secure Error Handling

Responses returned to clients must never expose internal PostgreSQL errors or database stack traces:

```csharp
// Error response contract
public record ApiErrorResponse(string Code, string Message, string? TraceId);

// Global Exception Handler
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new ApiErrorResponse(
            Code: "INTERNAL_SERVER_ERROR",
            Message: "An unexpected error occurred. Please contact the administrator.",
            TraceId: Activity.Current?.Id ?? context.TraceIdentifier
        ));
    });
});
```
