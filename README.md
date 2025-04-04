**C# Coding Standards สำหรับการพัฒนา .NET 8 Web API** 

เอกสารอ้างอิงในการทำ Code Review หรือสร้าง Coding Guideline ภายในองค์กร

#
## 🧠 **C# Coding Standards สำหรับพัฒนา .NET 8 Web API (เวอร์ชันขยาย)**

#

### ✅ **1. Naming Conventions (มาตรฐานการตั้งชื่อ)**

การตั้งชื่อมีผลโดยตรงต่อความเข้าใจของโค้ด ใช้ **ชื่อที่สื่อความหมาย**, **สั้นแต่ชัดเจน**, และ **สม่ำเสมอ**

| ชนิดของโค้ด | รูปแบบ | ตัวอย่าง |
|-------------|--------|----------|
| Class / Interface | PascalCase | `ProductService`, `IProductRepository` |
| Method / Property | PascalCase | `CalculateTotalPrice()`, `ProductName` |
| Local Variable | camelCase | `productList`, `discountRate` |
| Private Field | _camelCase | `_context`, `_logger` |
| Constant | PascalCase | `DefaultPageSize`, `MaxRetryCount` |
| Async Method | ลงท้ายด้วย `Async` | `GetByIdAsync()`, `SaveChangesAsync()` |

📌 **ตัวอย่าง:**
```csharp
public class ProductService : IProductService
{
    private readonly IProductRepository _productRepository;
    private const int DefaultPageSize = 10;

    public async Task<ProductDto> GetByIdAsync(int id)
    {
        var product = await _productRepository.GetByIdAsync(id);
        return product is null ? null : new ProductDto(product);
    }
}
```

#

### ✅ **2. Solution & Project Structure (โครงสร้างโปรเจกต์)**

ใช้ Clean Architecture หรืออย่างน้อยแยก Layer ชัดเจน (Controller, Service, Repository):

```
ProductManagement/
├── src/
│   ├── ProductManagement.API               ← Web API
│   ├── ProductManagement.Application       ← Use Case Logic
│   ├── ProductManagement.Domain            ← Entities, Interfaces
│   └── ProductManagement.Infrastructure    ← DB, External Services
└── tests/
    └── ProductManagement.Tests             ← Unit Tests
```

#

### ✅ **3. API Controller Standards (มาตรฐานการเขียน Controller)**

- ใช้ `[ApiController]` และ `[Route("api/[controller]")]`
- ไม่เขียน Logic ธุรกิจใน Controller
- ควรใช้ `ActionResult<T>` เสมอ
- ตรวจสอบ Model ด้วย `ModelState.IsValid`

📌 **ตัวอย่าง:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductService productService, ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var result = await _productService.GetByIdAsync(id);
        if (result is null)
        {
            _logger.LogWarning("Product not found: {Id}", id);
            return NotFound();
        }
        return Ok(result);
    }

    [HttpPost]
    public async Task<ActionResult> CreateProduct([FromBody] CreateProductRequest request)
    {
        if (!ModelState.IsValid) return BadRequest(ModelState);

        var productId = await _productService.CreateAsync(request);
        return CreatedAtAction(nameof(GetProduct), new { id = productId }, null);
    }
}
```

#

### ✅ **4. Exception Handling (การจัดการข้อผิดพลาด)**

- ใช้ **Global Exception Middleware**
- ไม่ควรใช้ `try-catch` ทั่วไปในทุก Method
- ใช้ `ILogger` เพื่อล็อกข้อผิดพลาด

📌 **ตัวอย่าง Middleware:**
```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An unhandled exception occurred.");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { error = "Internal Server Error" });
        }
    }
}
```

#

### ✅ **5. Logging Standards (มาตรฐานการ Log)**

- ใช้ `ILogger<T>` จาก DI
- ระบุ Context และไม่ log ข้อมูล Sensitive เช่น Password
- ใช้ `LogInformation`, `LogWarning`, `LogError` อย่างเหมาะสม

📌 **ตัวอย่าง:**
```csharp
_logger.LogInformation("Fetching product with ID: {ProductId}", productId);
_logger.LogWarning("Product with ID {ProductId} not found", productId);
_logger.LogError(ex, "Error occurred while getting product: {ProductId}", productId);
```

#

### ✅ **6. Dependency Injection (DI)**

- ลงทะเบียน Service ทั้งหมดผ่าน `Program.cs`
- ใช้ `Scoped` สำหรับ Business Logic / Repository
- หลีกเลี่ยง `new` ภายใน class

📌 **ตัวอย่างใน `Program.cs`:**
```csharp
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();
```

---

### ✅ **7. Async/Await Best Practices**

- ใช้ `async/await` ทุกครั้งที่มี I/O หรือ DB Access
- หลีกเลี่ยง `.Result` หรือ `.Wait()` ที่อาจ Deadlock
- ใช้ชื่อ Method ลงท้ายด้วย `Async`

📌 **ตัวอย่าง:**
```csharp
public async Task<List<Product>> GetAllAsync()
{
    return await _dbContext.Products.ToListAsync();
}
```

#

### ✅ **8. Clean Method Design**

- ไม่ควรมี method ที่ยาวเกิน 30 บรรทัด
- แยก method ย่อยเมื่อมีเงื่อนไขซับซ้อน
- ใช้ชื่อ method ให้สื่อถึงพฤติกรรม

📌 **ตัวอย่าง:**
```csharp
public async Task<decimal> CalculateDiscountedPriceAsync(int productId)
{
    var product = await _productRepository.GetByIdAsync(productId);
    if (product is null) throw new NotFoundException("Product not found");

    var discount = await _discountService.GetDiscountForProductAsync(productId);
    return ApplyDiscount(product.Price, discount);
}

private decimal ApplyDiscount(decimal price, decimal discount)
{
    return price - (price * discount);
}
```

#

### ✅ **9. Unit Testing**

- ใช้ `xUnit` + `FluentAssertions`
- แยก Arrange, Act, Assert ให้ชัดเจน
- ชื่อ Method สื่อความหมาย: `MethodName_State_ExpectedResult`

📌 **ตัวอย่าง:**
```csharp
[Fact]
public async Task GetByIdAsync_WithValidId_ReturnsProduct()
{
    // Arrange
    var productId = 1;
    var expected = new Product { Id = 1, Name = "Test Product" };
    _repoMock.Setup(x => x.GetByIdAsync(productId)).ReturnsAsync(expected);

    // Act
    var result = await _service.GetByIdAsync(productId);

    // Assert
    result.Should().NotBeNull();
    result.Id.Should().Be(productId);
}
```

#

### ✅ **10. Security Standards**

- ใช้ `[FromBody]`, `[FromQuery]`, `[FromRoute]` ให้ชัดเจน
- ใช้ `[Authorize]` กับ Endpoint ที่ต้องการสิทธิ์
- Validate Input ทุกรูปแบบ

📌 **ตัวอย่าง:**
```csharp
[Authorize(Roles = "Admin")]
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateProductRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    var id = await _productService.CreateAsync(request);
    return CreatedAtAction(nameof(GetProduct), new { id }, null);
}
```

#

### ✅ **11. Dapper/EF Core Standards**

- ไม่เขียน SQL ใน Controller
- แยก Logic ไปไว้ใน Repository
- ใช้ DTO ในการ Return ไม่ Return Entity ตรงๆ

📌 **ตัวอย่าง (Dapper):**
```csharp
public async Task<ProductDto> GetByIdAsync(int id)
{
    const string sql = "SELECT Id, Name, Price FROM Products WHERE Id = :Id";
    using var connection = _context.CreateConnection();
    return await connection.QueryFirstOrDefaultAsync<ProductDto>(sql, new { Id = id });
}
```

#

### ✅ **12. Code Style & Clean Code**

- ใช้ `var` กับ type ชัดเจนเท่านั้น
- ใส่ `{}` ทุกครั้งใน `if`, `for`, `foreach`
- ใช้ `nameof()` แทน string literal
- ไม่ hardcode ค่าใน Logic

📌 **ตัวอย่าง:**
```csharp
if (order.Status == OrderStatus.Pending)
{
    _logger.LogInformation("Order {OrderId} is pending", order.Id);
}
```

#

## ✨ สรุป Checklist สำหรับทีมพัฒนา

✅ ตั้งชื่อให้สื่อความหมาย  
✅ แยก Layer ตาม Clean Architecture  
✅ ไม่เขียน Logic ธุรกิจใน Controller  
✅ ใช้ `ILogger` แทน `Console.WriteLine`  
✅ ใช้ `async/await` กับ I/O ทุกชนิด  
✅ ใช้ Global Exception Middleware  
✅ ทำ Unit Test ให้ครอบคลุม Business Logic  
✅ Validate Model ทุกรูปแบบก่อนใช้งาน  
✅ แยก Entity/DTO อย่างชัดเจน  
✅ เขียนโค้ดให้คนอ่านง่ายกว่าคอมพิวเตอร์

#
