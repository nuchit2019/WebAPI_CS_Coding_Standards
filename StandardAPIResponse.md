การออกแบบ **รูปแบบ Response ที่เป็นมาตรฐาน** (Standard API Response) จะช่วยให้ Frontend ใช้งาน API ได้ง่ายขึ้น, ลดความสับสนในการ Debug และทำให้ Logging/Monitoring มีประสิทธิภาพมากขึ้น

---

## ✅ รูปแบบ **Standard API Response Format** (เวอร์ชันแนะนำสำหรับ .NET 8 Web API)

### 📦 **1. Success Response**

```json
{
  "success": true,
  "message": "Product retrieved successfully.",
  "data": {
    "id": 1,
    "name": "iPhone 15",
    "price": 45999
  },
  "errors": null
}
```

### 📦 **2. Error Response**

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": null,
  "errors": [
    {
      "field": "name",
      "message": "Name is required."
    },
    {
      "field": "price",
      "message": "Price must be greater than 0."
    }
  ]
}
```

---

## 🧱 โครงสร้าง Object

```csharp
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public string Message { get; set; } = string.Empty;
    public T? Data { get; set; }
    public List<ApiError>? Errors { get; set; }
}

public class ApiError
{
    public string Field { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
}
```

---

## 🛠 ตัวอย่างการใช้งานใน Controller

### 🎯 Success (200 OK)
```csharp
[HttpGet("{id}")]
public async Task<ActionResult<ApiResponse<ProductDto>>> GetById(int id)
{
    var product = await _service.GetByIdAsync(id);
    if (product == null)
    {
        return NotFound(new ApiResponse<ProductDto>
        {
            Success = false,
            Message = "Product not found"
        });
    }

    return Ok(new ApiResponse<ProductDto>
    {
        Success = true,
        Message = "Product retrieved successfully.",
        Data = product
    });
}
```

---

### 🎯 Validation Error (400 Bad Request)
```csharp
[HttpPost]
public async Task<ActionResult<ApiResponse<object>>> Create([FromBody] ProductCreateRequest request)
{
    if (!ModelState.IsValid)
    {
        var errors = ModelState
            .Where(e => e.Value?.Errors.Count > 0)
            .Select(e => new ApiError
            {
                Field = e.Key,
                Message = e.Value?.Errors.First().ErrorMessage ?? ""
            })
            .ToList();

        return BadRequest(new ApiResponse<object>
        {
            Success = false,
            Message = "Validation failed.",
            Errors = errors
        });
    }

    // Proceed with create...
}
```

---

## 🎨 รูปแบบ HTTP Status + Message ที่ควรใช้

| HTTP Status | ใช้เมื่อ | รูปแบบ Message |
|-------------|-----------|----------------|
| `200 OK` | สำเร็จ | `Product retrieved successfully.` |
| `201 Created` | สร้างสำเร็จ | `Product created successfully.` |
| `204 No Content` | ลบสำเร็จ / ไม่มีผลลัพธ์ | `Product deleted.` |
| `400 Bad Request` | Validation / Input ผิด | `Validation failed.` |
| `401 Unauthorized` | ยังไม่ได้ล็อกอิน | `Unauthorized request.` |
| `403 Forbidden` | ไม่มีสิทธิ์เข้าถึง | `You do not have permission.` |
| `404 Not Found` | ไม่พบข้อมูล | `Product not found.` |
| `500 Internal Server Error` | เกิดข้อผิดพลาดภายใน | `An unexpected error occurred.` |

---

## 🧪 ทำให้ Response เป็นมาตรฐานทุก Endpoint (Middleware)

สามารถสร้าง `ResultFilter` หรือ `ActionFilter` เพื่อ **Wrap Response อัตโนมัติ**

หรือใช้ **Custom IActionResult** เช่น:

```csharp
public class ApiOkResult<T> : IActionResult
{
    private readonly T _data;
    private readonly string _message;

    public ApiOkResult(T data, string message = "Success")
    {
        _data = data;
        _message = message;
    }

    public async Task ExecuteResultAsync(ActionContext context)
    {
        var response = new ApiResponse<T>
        {
            Success = true,
            Message = _message,
            Data = _data
        };

        var objectResult = new OkObjectResult(response);
        await objectResult.ExecuteResultAsync(context);
    }
}
```

ใช้ใน Controller ได้แบบนี้:
```csharp
return new ApiOkResult(product, "Product fetched successfully");
```

---

## ✨ สรุป Best Practices

✅ ทุก Response ควรมี: `success`, `message`, `data`, `errors`  
✅ Error ทุกประเภทควรให้ Message ที่อ่านง่าย และอธิบายได้  
✅ Response Structure ต้อง **คงที่**, ง่ายต่อการแสดงผลและ Debug  
✅ แยก `ModelState` Error ให้ชัดเจน พร้อม Field ชี้ชัดเจน

---

ถ้าต้องการให้จัดทำเป็น **Library Class ช่วย Wrap Response**, หรือสร้าง Middleware ที่บังคับใช้มาตรฐานนี้ทั่วระบบ บอกได้เลยครับ เดี๋ยวจัดให้เต็มระบบ 💪
