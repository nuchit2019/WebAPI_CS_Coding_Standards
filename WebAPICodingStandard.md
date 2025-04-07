# **Web API Coding Standard** (มาตรฐานการเขียนโค้ด Web API) ที่ดี ชัดเจน และง่ายต่อการดูแลรักษา สำหรับการพัฒนา Web API ด้วย C# .NET:

---

## 🧩 1. Solution & Project Structure

โครงสร้าง Solution ควรเป็นระเบียบ ใช้แนวคิด Clean Architecture เช่น:

```
SolutionName
├── src
│   ├── SolutionName.API             # RESTful API Project (Controllers)
│   ├── SolutionName.Application     # Business Logic (Services, DTOs)
│   ├── SolutionName.Domain          # Core Entities & Domain logic
│   └── SolutionName.Infrastructure  # Database & External Service Connections
└── tests
    └── SolutionName.Tests           # Unit & Integration Tests
```

---

## 📦 2. Naming Conventions

### ✔️ General
- ชื่อ **Classes, Methods, Properties** ใช้ PascalCase เช่น:
```csharp
public class ProductService { }
public string GetProductById(int productId) { }
public string ProductName { get; set; }
```

### ✔️ Private fields ใช้ _camelCase
```csharp
private readonly IProductRepository _productRepository;
```

### ✔️ Interfaces นำหน้าด้วยตัว “I”
```csharp
public interface IProductRepository { }
```

### ✔️ Async Method เติม “Async” ต่อท้ายชื่อ method
```csharp
public async Task<Product> GetProductByIdAsync(int productId)
```

---

## 🚧 3. Controllers & Routing

### ✔️ คำแนะนำ
- Controllers ลงท้ายด้วย “Controller” เสมอ
- ใช้ `[ApiController]` Attribute
- ใช้ Routing แบบ `[Route("api/[controller]")]`

### ✔️ ตัวอย่างที่ดี
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetByIdAsync(int id)
    {
        // Logic
    }
}
```

---

## 📬 4. HTTP Verbs & Status Codes

ใช้ HTTP Verbs และ Status Codes ตามมาตรฐาน REST API อย่างเคร่งครัด เช่น:

| Action            | HTTP Verb | Status Code |
|-------------------|-----------|-------------|
| Create            | POST      | 201 Created |
| Read              | GET       | 200 OK      |
| Update (Full)     | PUT       | 200 OK      |
| Update (Partial)  | PATCH     | 200 OK      |
| Delete            | DELETE    | 204 No Content |
| Validation Error  | -         | 400 Bad Request |
| Unauthorized      | -         | 401 Unauthorized |
| Forbidden         | -         | 403 Forbidden |
| Not Found         | -         | 404 Not Found |
| Server Error      | -         | 500 Internal Server Error |

ตัวอย่างเช่น:

```csharp
[HttpPost]
public async Task<IActionResult> CreateAsync(CreateProductRequest request)
{
    var created = await _productService.CreateAsync(request);
    return CreatedAtAction(nameof(GetByIdAsync), new { id = created.Id }, created);
}
```

---

## 🛡️ 5. Model Validation & DTOs

- ใช้ Data Annotations หรือ FluentValidation
- ไม่ใช้ Domain Entities โดยตรงใน Controllers

```csharp
public class CreateProductRequest
{
    [Required]
    [StringLength(100)]
    public string Name { get; set; }
}
```

### ✔️ Validation ตัวอย่าง
```csharp
[HttpPost]
public IActionResult Create(CreateProductRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    // logic...
}
```

หรือให้ใช้ `[ApiController]` Attribute ที่จะตรวจสอบ ModelState อัตโนมัติและคืน `400 BadRequest`

---

## 🔧 6. Dependency Injection (DI)

- Inject dependency ผ่าน Constructor เท่านั้น
- ใช้ Interface เพื่อสนับสนุน Unit Testing

```csharp
private readonly IProductService _productService;

public ProductsController(IProductService productService)
{
    _productService = productService;
}
```

---

## 🧹 7. Error Handling & Logging

### ✔️ Error Handling

- ใช้ Global Exception Middleware
- ควรมี Response Wrapper มาตรฐาน
```csharp
app.UseMiddleware<GlobalExceptionMiddleware>();
```

### ✔️ Logging

- ใช้ `ILogger<T>` หรือ Library อย่าง `Serilog`

```csharp
_logger.LogInformation("Product created successfully: {@Product}", product);
```

---

## 📦 8. ORM/Data Access

- แยก Infrastructure Layer ชัดเจน
- ใช้ Repository Pattern หรือ DbContext แยกตามความรับผิดชอบ
- เน้น Query ประสิทธิภาพสูง เช่น Dapper หรือ EF Core

---

## 🚀 9. Asynchronous Programming

- ใช้ `async-await` อย่างถูกต้อง ไม่มี `.Wait()` หรือ `.Result`
- ใช้ `Task` หรือ `ValueTask` สำหรับ async method เสมอ

```csharp
public async Task<ProductDto> GetAsync(int id)
{
    return await _productRepository.GetAsync(id);
}
```

---

## 📚 10. Testing Standard

- แยก Test Project
- ใช้ xUnit, NUnit หรือ MSTest
- ครอบคลุม Unit และ Integration tests

ตัวอย่าง Unit Test:
```csharp
[Fact]
public async Task GetByIdAsync_WhenCalled_ReturnsProduct()
{
    // Arrange
    var mockRepo = new Mock<IProductRepository>();
    mockRepo.Setup(repo => repo.GetAsync(1))
            .ReturnsAsync(new Product { Id = 1 });

    var service = new ProductService(mockRepo.Object);

    // Act
    var product = await service.GetByIdAsync(1);

    // Assert
    Assert.NotNull(product);
    Assert.Equal(1, product.Id);
}
```

---

## 🗃️ 11. Documentation & Swagger

- ใช้ XML Comments ในโค้ด
- Integrate กับ Swagger/OpenAPI

```csharp
/// <summary>
/// Retrieves a specific product.
/// </summary>
/// <param name="id">Product ID</param>
/// <returns>Product details</returns>
[HttpGet("{id:int}")]
public IActionResult Get(int id) { ... }
```

---

## 🚦 12. Versioning API

- API ควรมีเวอร์ชั่นที่ชัดเจน
```csharp
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsController : ControllerBase
{
}
```

---

## 🗂️ 13. Response Standardization

- Response Format เป็นมาตรฐานเดียวกัน

```json
{
  "success": true,
  "message": "Product retrieved successfully",
  "data": { }
}
```

---

## 🚧 14. Security & Authorization

- ใช้ JWT หรือ OAuth
- ป้องกัน Sensitive Information
- ใช้ `[Authorize]` Attribute

```csharp
[Authorize(Roles = "Admin,Manager")]
[HttpDelete("{id:int}")]
public IActionResult Delete(int id) { }
```

---

## 📏 15. Formatting & Style Guide

- ใช้ .editorconfig เพื่อบังคับใช้ coding standard อัตโนมัติ
- ทุกคนในทีมใช้ formatting เดียวกัน

---

## 📑 สรุปมาตรฐานโดยย่อ

✅ Clean Architecture  
✅ Naming ชัดเจน  
✅ RESTful ตามมาตรฐาน  
✅ Dependency Injection  
✅ Async Programming ถูกต้อง  
✅ Global Error Handling  
✅ Logging เป็นระบบ  
✅ Unit Tests ครอบคลุม  
✅ Documentation & Swagger  
✅ API Versioning  
✅ Security  
✅ Response Wrapper มาตรฐาน  

---

การปฏิบัติตามมาตรฐานนี้จะช่วยให้ทีมพัฒนาโค้ดที่มีคุณภาพสูง ดูแลรักษาง่าย และต่อยอดในอนาคตได้อย่างยั่งยืน.
#
