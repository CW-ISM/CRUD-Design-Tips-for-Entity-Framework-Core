# CRUD Design Tips for Entity Framework Core: Expression Predicates, IQueryable & Partial Updates

## Introduction

Some useful tips I learned while working on my team's project.  
We'll cover four things:  
- [Using Expression Predicates for flexible queries](#section-1-expression-predicates)
- [Direct AppDbContext Usage vs Repository Pattern](#section-2-direct-_context-usage-vs-repository-pattern)
- [Understanding `IQueryable` vs `IEnumerable`](#section-3-iqueryable-vs-ienumerable)
- [Partial updates with DTOs and EF Core's change tracking](#section-4-partial-updates-using-dtos-and-ef-core-change-tracking)
- [User Model Implementation (For Context)](#user-model-implementation)
---


## Section 1. Using Expression Predicates for flexible queries

### What are Predicates?
A ***predicate*** is a function that returns a bool value based on some condition. It’s commonly used to filter data in LINQ queries.

Predicates are typically expressed as lambda functions (e.g., u => u.Username == "johndoe"), which are concise and easy to use. However, predicates can also be:
- **Named methods:** You can use regular methods instead of lambdas.
- **Method references:** You can pass a method directly as a reference.
- **Delegates:** In older C# code, you may see predicates as delegate types like `Predicate<T>`.

### Example of Predicate:
```csharp
// Lambda expression predicate
Expression<Func<User, bool>> predicate = u => u.Username == "johndoe";

// Named method predicate (alternative)
public bool IsJohnDoe(User u) => u.Username == "johndoe";
Expression<Func<User, bool>> predicateMethod = IsJohnDoe;

// Using method reference (alternative)
Expression<Func<User, bool>> predicateRef = IsJohnDoe;
```
<br/>
When querying your database, you often need to filter records based on conditions.

Using `Expression<Func<T, bool>` lets EF Core translate your filters directly into SQL queries, which is faster and more efficient.<br/>
This is different from just using a `Func<T, bool>`, which can’t be converted to SQL and runs in memory instead — slower and less efficient.


```csharp
// Uses Expression<Func<User, bool>> so EF Core can translate this to SQL
public User? GetSingleWhere(Expression<Func<User, bool>> predicate)
{
    return _context.Users.FirstOrDefault(predicate);
}

// Uses Func<User, bool>, which EF Core cannot translate to SQL and will execute in memory
public User?GetSingleWhere(Func<User, bool> predicate)
{
    return _context.Users.FirstOrDefault(predicate);
}
```
---

## Section 2. Direct AppDbcontext Usage vs Repository Pattern

Now, let’s contrast two approaches to querying your database:
- Using `_context` directly
- Using a repository for centralized data access.

```csharp
AppDbContext _context;

User? user1 = _context.Users.FirstOrDefault(u => u.Username == "johndoe");

User? user2 = _context.Product.FirstOrDefault(u => u.Username == "janedoe");
//                     ↑ Uses wrong table
```

### Using Repository Pattern (`UsersRepository`)

```csharp
// A centralized place for doing CRUD operations for Users table
public class UsersRepository
{
    private readonly AppDbContext _context;

    public UsersRepository(AppDbContext context)
    {
        _context = context;
    }

    // Safely gets User by username
    public User? GetByUsername(string username)
        => GetSingleWhere(u => u.Username == username);
    
    // Flexible GetSingle using WHERE clause
    public User? GetSingleWhere(Expression<Func<User, bool>> predicate)
    {
        return _context.Users.FirstOrDefault(predicate);
    }
}
```

---

## Section 3. IQueryable vs IEnumerable

Next, let’s talk about IQueryable and IEnumerable.</br>

### `IQueryable<T>`

- Represents a **deferred query** against the data source (like a database).  
- Query execution is **deferred** until you actually enumerate the results (e.g., `ToList()` or `foreach`).  
- Allows further filtering, sorting, and paging to be added before the query runs, all converted to SQL.  
- Great when you want flexible, composable queries.
> **Note:** Deferred meaning the query is not executed immediately

### `IEnumerable<T>`

- Represents an **in-memory collection**.  
- When you return `IEnumerable`, it usually means the query has **already executed** and results are loaded in memory.  
- Less flexible for composing queries, and potentially less efficient for large datasets.

### When to use each?

- Use `IQueryable` if you want to allow callers to build more complex queries before execution.  
- Use `IEnumerable` if you want to return a fully realized list right away.

### Example
```csharp
// Returns a List<User> after executing the query immediately
public List<User> GetWhereAsList(Expression<Func<User, bool>> predicate)
{
    return _context.Users.Where(predicate).ToList();
}

// Returns IQueryable<User>, allowing further composition of the query
public IQueryable<User> GetWhereAsQueryable(Expression<Func<User, bool>> predicate)
{
    return _context.Users.Where(predicate);
}

// Example of using GetWhereAsList, which executes the query immediately
var users = GetWhereAsList(u => u.Username.StartsWith("a"))
            .Where(u => u.FirstName == "John"); // This filtering happens in-memory

// Example of using GetWhereAsQueryable, which allows additional filtering before execution
var users = GetWhereAsQueryable(u => u.Username.StartsWith("a"))
            .Where(u => u.FirstName == "John")
            .ToList(); // Forces the query to execute at this point
```

---

## Section 4. Partial Updates using DTOs and EF Core Change Tracking

Updating entities can get tricky if you don't want to overwrite all properties, especially with large objects or PATCH-like updates.
> **Note:** Partial updates avoid overwriting unchanged fields, useful in PATCH scenarios or when updating only a few properties.

### Define a **Data Transfer Object (DTO)** that contains only the fields you want to update.  
A Data Transfer Object is an object used to pass around data between different software layers.

#### UserUpdateDTO model
```csharp
public class UserUpdateDTO
{
    // Make fields nullable
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public string? Email { get; set; }
    public string? Username { get; set; }

    public UserUpdateDTO(
        string? firstName = null,
        string? lastName = null,
        string? email = null,
        string? username = null
        )
    {
        FirstName = firstName;
        LastName = lastName;
        Email = email;
        Username = username;
    }

    public UserUpdateDTO() { }
}
```

---

### In your update method, **apply only non-null fields** from the DTO to the entity.  

```csharp
// Applies only non-null fields from DTO for a safe partial update
public bool Update(int id, UserUpdateDTO dto)
{
    User? user = _context.Users.Find(id);
    if (user == null) return false;

    user.FirstName = dto.FirstName ?? user.FirstName;
    user.LastName = dto.LastName ?? user.LastName;
    user.Email = dto.Email ?? user.Email;
    user.Username = dto.Username ?? user.Username;
    //              ↑ if this is not null it assigns user.Username
    //                to dto.Username otherwise keeps original value

    // EF Core tracks changes and updates only modified fields
    return _context.SaveChanges() > 0;
}
```

---

## Summary

- Use `Expression<Func<T, bool>>` for query predicates to enable EF Core to translate filters into efficient SQL.  
- Prefer returning `IQueryable<T>` when you want flexible, composable queries; use `IEnumerable<T>` when you want to return fully loaded data.  
- For updates, use nullable DTOs to apply only changed fields and leverage EF Core’s change tracking to minimize database writes.

---

### User Model Implementation
```csharp
public class User
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; } = 0;

    [Required]
    public string FirstName { get; set; } = string.Empty;

    [Required]
    public string LastName { get; set; } = string.Empty;

    [Required]
    public string Email { get; set; } = string.Empty;

    [Required]
    public string Username { get; set; } = string.Empty;

    public User(int id, string firstName, string lastName, string Email, string username)
    {
        Id = id;
        FirstName = firstName;
        LastName = lastName;
        Email = email;
        Username = username;
    }

    public User() { }
}
```

---