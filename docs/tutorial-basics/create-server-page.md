---
sidebar_position: 2
---

# Server-Side

## Introduction

This guide demonstrates how to create a complete module in the UW Platform using the Book Store example. We'll walk through creating a fully functional module with CRUD operations, proper validations, and multi-tenant support.

:::tip My tip

Make Sure to have followed the [client-side guide](/docs/tutorial-basics/create-client-page) to set up your development environment.

:::

## Prerequisites

Before starting module development, ensure you have:
- Access to the UW Platform solution
- SQL Server installed
- Entity Framework Core tools
- Required development environment setup

## Creating the Book Module

### Entity Creation

1. Create the Book entity in the Domain layer (`UW.Platform.Domain.Entities`):

```csharp
public class Book : AuditableEntityBase, IMultiTenantEntity
{
    [MaxLength(StringLength.L50)]
    public string Name { get; set; } = null!;
    public BookTypeEnum Type { get; set; }
    public DateTime PublishDate { get; set; }
    public decimal Price { get; set; }
    public int? TenantId { get; set; }
    public Tenant? Tenant { get; set; }
}
```

2. Create the BookTypeEnum in `UW.Platform.Domain.Enums`:

```csharp
public enum BookTypeEnum
{
    [Description("Undefined")]
    Undefined,
    [Description("Adventure")]
    Adventure,
    [Description("Biography")]
    Biography,
    // ... other types
}
```

### Database Configuration

1. Add Book entity to `IAppDbContext`:

```csharp
public interface IAppDbContext
{
    public DbSet<Book> Books { get; set; }
    // ... other entities
}
```

2. Add entity configuration (optional):

```csharp
public class BookConfiguration : IEntityTypeConfiguration<Book>
{
    public void Configure(EntityTypeBuilder<Book> builder)
    {
        builder.HasIndex(x => x.Name)
            .IsUnique(false);
    }
}
```

3. Create and apply migration:

```bash
dotnet ef migrations add Create_Book_Entity --project src/UW.Platform.Infrastructure --startup-project src/WebApps/UW.Platform.HostApi --context AppDbContext --output-dir Data/Migrations
```

### Permission Setup

1. Add permissions to `TenantPermissions.cs`:

```csharp
public static class TenantPermissions
{
    [CustomDisplayName("Books Management")]
    public static class Books
    {
        [CustomIdentity(2451)]
        [CustomDisplayName("Read Book list and details")]
        public const string Default = $"{GroupName}.Books";

        [CustomIdentity(2452)]
        [CustomDisplayName("Create Book")]
        public const string Create = $"{Default}.Create";

        [CustomIdentity(2453)]
        [CustomDisplayName("Update Book")]
        public const string Update = $"{Default}.Update";

        [CustomIdentity(2454)]
        [CustomDisplayName("Delete Book")]
        public const string Delete = $"{Default}.Delete";
    }

    public static List<string> GetBookPermissions()
    {
        return [Books.Default, Books.Create, Books.Update, Books.Delete];
    }
}
```

### API Endpoints

Create the following endpoint structure in `UW.Platform.TenantApi.UseCases`:

```
Book/
├── CreateBook/
│   ├── Endpoint.cs
│   ├── Handler.cs
│   ├── Mapper.cs
│   └── Validator.cs
├── GetBook/
│   ├── Endpoint.cs
│   └── Handler.cs
├── GetBookList/
│   ├── Endpoint.cs
│   └── Handler.cs
├── UpdateBook/
│   ├── Endpoint.cs
│   ├── Handler.cs
│   └── Validator.cs
└── DeleteBook/
    ├── Endpoint.cs
    └── Handler.cs
```

Example implementation for CreateBook endpoint:

1. Endpoint.cs:
```csharp
public class Endpoint : Endpoint<Request, string>
{
    public override void Configure()
    {
        Post("books");
        Permissions(TenantPermissions.Books.Create);
    }

    public override async Task HandleAsync(Request req, CancellationToken ct)
    {
        var response = await handler.HandleAsync(req, ct);
        await SendAsync(Utils.Encrypt(response), cancellation: ct);
    }
}

public class Request : IEncryptedDto
{
    public string Name { get; set; } = null!;
    public string Type { get; set; } = null!;
    public DateTime PublishDate { get; set; }
    public decimal Price { get; set; }
}
```

2. Handler.cs:
```csharp
public class Handler : IScopedHandler
{
    public async Task<int> HandleAsync(Request req, CancellationToken ct = default)
    {
        var validation = ValidationContext.Instance;
        var type = Enum.TryParse<BookTypeEnum>(req.Type, out var validType);
        if (!type)
            validation.ThrowError(l["ErrorMessage.InvalidBookType"]);

        var book = mapper.ToEntity(req);
        book.Type = validType;

        dbContext.Books.Add(book);
        await dbContext.SaveChangesAsync(ct);
        return book.Id;
    }
}
```

## Localization

Add localizations in `UW.Platform.Infrastructure.Data.DataSeed.Localizations`:

1. ar.json:
```json
{
  "ErrorMessage.BookNotExist": "الكتاب غير موجودة",
  "ErrorMessage.InvalidBookType": "خطأ بنوع الكتاب"
}
```

2. en.json:
```json
{
  "ErrorMessage.BookNotExist": "Book Not Exist",
  "ErrorMessage.InvalidBookType": "Invalid Book Type"
}
```

## Adding Feature to System

1. Update `AppFeatures.cs`:

```csharp
public static class AppFeatures
{
    [CustomIdentity(16)]
    [CustomDisplayName("Books Feature")]
    [Description("Books Feature")]
    public const string Books = "Books";
}
```

2. Update `SeedData.cs` to include the new feature and roles:

```csharp
public class SeedData
{
    private const string BooksRole = "Books Full Access";
    
    private static void PopulateDefaultTenantRoles(AppDbContext context)
    {
        if (!context.Roles.Any(u => u.NormalizedName == BooksRole.ToUpper()))
        {
            context.Roles.Add(new Role
            {
                Name = BooksRole,
                NormalizedName = BooksRole.ToUpper(),
                DisplayName = "Books Role Full Access",
                Description = "Can do Books actions like read, create, update and delete",
                RoleType = ApplicationTypeEnum.Tenant,
                RolePermissions = TenantPermissions.GetBookPermissions()
                    .Select(key => new RolePermission { 
                        PermissionId = allTenantPermissions[key].Id 
                    })
                    .ToList()
            });
        }
    }
}
```
