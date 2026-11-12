# BÁO CÁO THỰC HÀNH: XÂY DỰNG HỆ THỐNG PHÂN TÁN VỚI ASP.NET CORE MINIMAL API & HTTPCLIENT

* **Sinh viên thực hiện:** Cấn Văn Thái
* **Mã sinh viên:** DTC245200867 — **Lớp:** CNTTK23C
* **Môn học:** Phát triển ứng dụng phân tán

---

## Bước 1: Khởi tạo Solution và kiến trúc hai tiến trình độc lập

Hệ thống được tổ chức thành một Solution (`CatalogApi.slnx`) chứa hai dự án tách biệt nằm trong thư mục `src/`:
1. **`Catalog.Api` (Máy chủ):** Ứng dụng ASP.NET Core Empty lắng nghe yêu cầu HTTP trên cổng `5100`.
2. **`Catalog.Client` (Máy khách):** Ứng dụng Console đóng vai trò tiến trình độc lập, giao tiếp với máy chủ hoàn toàn qua giao thức HTTP/REST.

![Minh chứng cấu trúc thư mục và build thành công](images/01-cau-truc.png)

---

## Bước 2 – 5: Xây dựng các Endpoint phía Máy chủ (`Catalog.Api`)

Dữ liệu được lưu trữ tạm thời trong bộ nhớ RAM (`List<ProductDto>`) với 3 sản phẩm mẫu khởi tạo sẵn. Hệ thống tách biệt hợp đồng dữ liệu đầu vào (`CreateProductRequest`) và đầu ra (`ProductDto`), đồng thời chuẩn hóa định dạng lỗi (`ApiError`).

### Bảng tổng hợp kiểm thử qua `curl.exe` (Bước 3 – 5)

| STT | Ca kiểm thử | Phương thức & URI | Mã trạng thái | Thành phần xử lý & Giải thích |
| :-: | :--- | :--- | :---: | :--- |
| 1 | Kiểm tra trạng thái máy chủ | `GET /` | `200 OK` | Handler trả về JSON trạng thái dịch vụ. |
| 2 | Lấy danh sách sản phẩm | `GET /api/products` | `200 OK` | Handler trả về mảng JSON gồm 3 sản phẩm mẫu. |
| 3 | Lấy chi tiết sản phẩm tồn tại | `GET /api/products/1` | `200 OK` | Handler tìm thấy sản phẩm có `Id = 1`. |
| 4 | Lấy chi tiết sản phẩm không tồn tại | `GET /api/products/999` | `404 Not Found` | Handler chạy, không tìm thấy nên trả về `404` kèm `ApiError`. |
| 5 | Sai kiểu tham số tuyến (`:int`) | `GET /api/products/abc` | `404 Not Found` | Framework chặn vì `abc` không phải số nguyên (body rỗng). |
| 6 | Gọi sai phương thức HTTP | `POST /api/products` *(chưa cài đặt)* | `405 Method Not Allowed` | Framework chặn vì URI tồn tại nhưng chưa hỗ trợ `POST`. |
| 7 | Tạo sản phẩm hợp lệ | `POST /api/products` (`sp-hop-le.json`) | `201 Created` | Handler tự sinh `Id = 4`, lưu vào RAM và trả kèm header `Location`. |
| 8 | Tạo sản phẩm vi phạm nghiệp vụ | `POST /api/products` (`sp-sai.json`) | `400 Bad Request` | Handler kiểm tra biên phát hiện giá âm/tên rỗng, trả về `INVALID_PRODUCT`. |
| 9 | Gửi thiếu header định dạng | `POST /api/products` *(thiếu Content-Type)* | `415 Unsupported Media Type` | Framework từ chối đọc body vì không xác định được định dạng JSON. |

![Minh chứng kết quả chạy curl](images/02-curl-test.png)

---

## Bước 6: Kiểm thử tập trung với tệp `requests.http`

Toàn bộ 8 kịch bản kiểm thử được gom vào tệp `requests.http` đặt tại thư mục gốc `CatalogApi` sử dụng biến môi trường `@api = http://localhost:5100`. Tất cả các ca đều trả về đúng mã trạng thái mong đợi.

![Minh chứng chạy 8 ca test trong requests.http](images/03-requests-http.png)

---

## Bước 7 – 8: Xây dựng Máy khách (`Catalog.Client`) và chạy kịch bản 4 nấc

Máy khách sử dụng `HttpClient` trỏ tới địa chỉ `BaseAddress` (nhận từ tham số dòng lệnh `args[0]` hoặc mặc định `http://localhost:5100`) và khai báo bản sao các `record` DTO để ánh xạ JSON.

Khi chạy đồng thời cả hai tiến trình, máy khách thực hiện tuần tự 4 nấc:
1. **Nấc 1:** Gọi `GET /api/products` và hiển thị danh sách sản phẩm hiện có.
2. **Nấc 2:** Gửi `POST /api/products` tạo mới sản phẩm `"Nhap mon REST API"`, kiểm tra mã `201 Created` trước khi đọc body và header `Location`.
3. **Nấc 3:** Gọi `GET /api/products/1` đọc lại chi tiết sản phẩm đầu tiên.
4. **Nấc 4:** Cố tình gửi dữ liệu sai (`CreateProductRequest("", "", -1)`), bắt mã `400 BadRequest` và in thông báo lỗi `INVALID_PRODUCT` an toàn mà không sập chương trình.

![Minh chứng chạy Catalog.Client đủ 4 nấc](images/04-client-4-nac.png)

---

## Bước 9: Bốn thí nghiệm về đặc trưng hệ thống phân tán

| Thí nghiệm | Thao tác thực hiện | Kết quả quan sát thực tế | Bài học rút ra về hệ phân tán |
| :---: | :--- | :--- | :--- |
| **1** | Chạy máy khách lần 2 (máy chủ vẫn đang bật) | Danh sách ban đầu đã có **4 sản phẩm**, chương trình tiếp tục tạo mới ra sản phẩm số **`[5]`** (`Location: /api/products/5`). | Trạng thái (dữ liệu) nằm ở máy chủ, không nằm ở máy khách. |
| **2** | Tắt máy chủ (`Ctrl + C`), sau đó chạy máy khách | Máy khách văng ngoại lệ: `Unhandled exception. HttpRequestException: No connection could be made... (localhost:5100)`. | Lỗi đặc trưng của hệ phân tán: tiến trình bên kia có thể vắng mặt bất cứ lúc nào. |
| **3** | Khởi động lại máy chủ và chạy máy khách | Danh sách quay trở lại **chỉ còn 3 sản phẩm** mặc định ban đầu (các sản phẩm 4, 5 đã biến mất). | Dữ liệu lưu tạm trong bộ nhớ RAM sẽ mất sạch khi tiến trình máy chủ dừng. |
| **4** | Chạy máy khách trỏ sang cổng sai (`http://localhost:5999`) | Máy khách báo lỗi từ chối kết nối tại cổng không tồn tại (`Connection refused (localhost:5999)`). | Hai bên phải thống nhất chính xác địa chỉ URL và cổng kết nối. |

![Minh chứng kết quả 4 thí nghiệm Bước 9](images/05-thi-nghiem-9.png)

---

## Bước 10: Lưu trữ lịch sử Git và Phiếu tự kiểm tra

### 1. Lịch sử Git
Mã nguồn được quản lý bằng Git với tệp `.gitignore` loại bỏ hoàn toàn các thư mục biên dịch tạm (`bin/`, `obj/`).

![Minh chứng lịch sử git log và git status](images/06-git-log.png)

### 2. Bảng tự đánh giá kết quả thực hành

| # | Yêu cầu kiểm tra | Kết quả mong đợi | Tự đánh giá |
| :-: | :--- | :--- | :---: |
| 1 | `GET /` | `200` | ✓ |
| 2 | `GET /api/products` | `200`, 3 phần tử | ✓ |
| 3 | `GET /api/products/1` | `200` | ✓ |
| 4 | `GET /api/products/999` | `404 + ApiError` | ✓ |
| 5 | `GET /api/products/abc` | `404` rỗng | ✓ |
| 6 | `POST` hợp lệ | `201 + Location` | ✓ |
| 7 | `POST` giá âm | `400 + ApiError` | ✓ |
| 8 | `POST` thiếu `Content-Type` | `415` | ✓ |
| 9 | Máy khách chạy | Đủ 4 nấc | ✓ |
| 10 | Tắt máy chủ, chạy máy khách | Lỗi kết nối (`Connection refused`) | ✓ |

---

## Phụ lục: Mã nguồn hoàn chỉnh

### Phụ lục A: `src/Catalog.Api/Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var products = new List<ProductDto>
{
    new(1, "Giao trinh Lap trinh phan tan", "Giao trinh chinh thuc", 150000),
    new(2, "Tai lieu RESTful API", "Huong dan thiet ke API", 90000),
    new(3, "Bai tap thuc hanh .NET", "Bo bai tap kem loi giai", 120000)
};

app.MapGet("/", () => Results.Ok(new
{
    service = "Catalog.Api",
    status = "running"
}));

app.MapGet("/api/products", () => Results.Ok(products));

app.MapGet("/api/products/{id:int}", (int id) =>
{
    var product = products.FirstOrDefault(p => p.Id == id);
    return product is null
        ? Results.NotFound(new ApiError("PRODUCT_NOT_FOUND", "Khong tim thay san pham."))
        : Results.Ok(product);
});

app.MapPost("/api/products", (CreateProductRequest request) =>
{
    if (string.IsNullOrWhiteSpace(request.Name) || request.Price <= 0)
    {
        return Results.BadRequest(new ApiError(
            "INVALID_PRODUCT",
            "Ten san pham khong duoc de trong va gia phai lon hon 0."));
    }

    var product = new ProductDto(
        Id: products.Max(p => p.Id) + 1,
        Name: request.Name.Trim(),
        Description: request.Description?.Trim() ?? "",
        Price: request.Price);

    products.Add(product);
    return Results.Created($"/api/products/{product.Id}", product);
});

app.Run();

record ProductDto(int Id, string Name, string Description, decimal Price);
record CreateProductRequest(string Name, string? Description, decimal Price);
record ApiError(string Code, string Message);
using System.Net;
using System.Net.Http.Json;

var apiBaseUrl = args.Length > 0 ? args[0] : "http://localhost:5100";
using var http = new HttpClient {
    BaseAddress = new Uri(apiBaseUrl)
};

var products = await http.GetFromJsonAsync<List<ProductDto>>("/api/products");
Console.WriteLine("Danh sach san pham:");
foreach (var p in products ?? []) {
    Console.WriteLine($" [{p.Id}] {p.Name} -{p.Price:N0} VND");
}

var createRequest = new CreateProductRequest(
    "Nhap mon REST API", "Sach tham khao", 120000);

var createResponse = await http.PostAsJsonAsync("/api/products", createRequest);

if (createResponse.StatusCode == HttpStatusCode.Created) {
    var created = await createResponse.Content.ReadFromJsonAsync<ProductDto>();
    Console.WriteLine($"\nDa tao san pham moi: [{created?.Id}] {created?.Name}");
    Console.WriteLine($"Doc lai tai: {createResponse.Headers.Location}");
} else {
    var error = await createResponse.Content.ReadFromJsonAsync<ApiError>();
    Console.WriteLine($"\nLoi: {error?.Code} - {error?.Message}");
    return;
}

var detail = await http.GetFromJsonAsync<ProductDto>("/api/products/1");
Console.WriteLine($"\nChi tiet san pham 1: {detail?.Name}");

var invalidResponse = await http.PostAsJsonAsync("/api/products",
    new CreateProductRequest("", "", -1));

Console.WriteLine($"\nThu gui du lieu sai: {(int)invalidResponse.StatusCode} {invalidResponse.StatusCode}");
var invalidError = await invalidResponse.Content.ReadFromJsonAsync<ApiError>();
Console.WriteLine($"{invalidError?.Code}: {invalidError?.Message}");

record ProductDto(int Id, string Name, string Description, decimal Price);
record CreateProductRequest(string Name, string? Description, decimal Price);
record ApiError(string Code, string Message);
