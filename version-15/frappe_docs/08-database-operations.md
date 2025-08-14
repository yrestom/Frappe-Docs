# Database Operations - Complete Guide

> **Comprehensive reference for ORM patterns, query building, schema management, and advanced database operations**

## Table of Contents

- [Database Architecture Overview](#database-architecture-overview)
- [ORM and Document Pattern](#orm-and-document-pattern)
- [Database Connection Management](#database-connection-management)
- [Query Building and Execution](#query-building-and-execution)
- [Schema Management](#schema-management)
- [Advanced Query Operations](#advanced-query-operations)
- [Transaction Management](#transaction-management)
- [Performance Optimization](#performance-optimization)
- [Database Utilities](#database-utilities)
- [Migration and Schema Changes](#migration-and-schema-changes)
- [Raw SQL Operations](#raw-sql-operations)
- [Best Practices](#best-practices)

## Database Architecture Overview

### Database Abstraction Layer

Based on analysis of `frappe/database/database.py:54-100`, Frappe implements a sophisticated database abstraction layer:

```python
class Database:
    """Main database interface supporting MySQL/MariaDB and PostgreSQL"""
    
    VARCHAR_LEN = 140
    MAX_COLUMN_LENGTH = 64
    
    # Standard table structure
    DEFAULT_COLUMNS = ("name", "creation", "modified", "modified_by", "owner", "docstatus", "idx")
    CHILD_TABLE_COLUMNS = ("parent", "parenttype", "parentfield")
    OPTIONAL_COLUMNS = ("_user_tags", "_comments", "_assign", "_liked_by")
    
    def __init__(self, host=None, user=None, password=None, port=None, cur_db_name=None):
        self.setup_type_map()
        self.transaction_writes = 0
        self.value_cache = {}
        self.before_commit = CallbackManager()
        self.after_commit = CallbackManager()
```

### Multi-Database Support

```python
# Database-specific implementations
from frappe.database.mariadb.database import MariaDBDatabase
from frappe.database.postgres.database import PostgresDatabase

# Query builder for different databases
from frappe.query_builder.builder import MariaDB, Postgres

# Usage based on database type
if frappe.conf.db_type == "postgres":
    frappe.qb = Postgres
else:
    frappe.qb = MariaDB  # Default MariaDB/MySQL
```

### Table Naming Convention

```python
def get_table_name(doctype):
    """Convert DocType name to table name"""
    return f"tab{doctype}"

# Examples:
# "User" -> "tabUser"
# "Sales Order" -> "tabSales Order"
# "Sales Order Item" -> "tabSales Order Item"
```

## ORM and Document Pattern

### Document-Based ORM

Based on analysis of `frappe/model/document.py:90-100`, Frappe uses a document-based ORM pattern:

```python
class LibraryBook(Document):
    """Custom DocType controller extending base Document"""
    
    # Type hints for auto-generated fields
    book_name: str
    isbn: str
    author: str
    publication_year: int
    status: str
    
    def validate(self):
        """Custom validation before save"""
        self.validate_isbn()
        self.set_book_code()
        
    def validate_isbn(self):
        """Validate ISBN format"""
        if not self.isbn:
            return
            
        # Remove hyphens and spaces
        isbn = re.sub(r'[-\s]', '', self.isbn)
        
        if not re.match(r'^\d{13}$', isbn):
            frappe.throw(_("ISBN must be 13 digits"))
            
        # Calculate check digit
        check_sum = sum(int(digit) * (1 if i % 2 == 0 else 3) 
                       for i, digit in enumerate(isbn[:-1]))
        
        check_digit = (10 - (check_sum % 10)) % 10
        if int(isbn[-1]) != check_digit:
            frappe.throw(_("Invalid ISBN check digit"))
            
    def set_book_code(self):
        """Auto-generate book code if not set"""
        if not self.book_code:
            # Generate code from first 3 letters of book name + year
            name_part = ''.join(c for c in self.book_name[:3] if c.isalpha()).upper()
            year_part = str(self.publication_year)[-2:] if self.publication_year else "00"
            
            # Find next sequence number
            existing_codes = frappe.get_all("Library Book",
                filters={"book_code": ["like", f"{name_part}{year_part}%"]},
                pluck="book_code"
            )
            
            sequence = len(existing_codes) + 1
            self.book_code = f"{name_part}{year_part}{sequence:03d}"
            
    def before_insert(self):
        """Hook called before document insertion"""
        self.log_activity("Created")
        
    def after_insert(self):
        """Hook called after document insertion"""
        self.update_inventory_count()
        self.send_notification("Book Added")
        
    def on_update(self):
        """Hook called after document update"""
        if self.has_value_changed("status"):
            self.log_status_change()
            
    def before_save(self):
        """Hook called before every save (insert/update)"""
        self.modified_by = frappe.session.user
        
    def on_submit(self):
        """Hook called when document is submitted"""
        if self.status != "Available":
            frappe.throw(_("Cannot submit book that is not available"))
        self.status = "Active"
        
    def on_cancel(self):
        """Hook called when document is cancelled"""
        self.status = "Cancelled"
        
    def on_trash(self):
        """Hook called before document deletion"""
        # Check if book has any active issues
        active_issues = frappe.db.exists("Library Issue", {
            "book": self.name,
            "return_date": ["is", "not set"]
        })
        
        if active_issues:
            frappe.throw(_("Cannot delete book with active issues"))
```

### Document Creation and Retrieval

```python
def create_library_book():
    """Create new library book document"""
    
    # Method 1: Using get_doc with dictionary
    book = frappe.get_doc({
        "doctype": "Library Book",
        "book_name": "Advanced Python Programming",
        "author": "John Doe", 
        "isbn": "978-0-123456-78-9",
        "publication_year": 2023,
        "status": "Available",
        "category": "Programming"
    })
    book.insert()
    
    # Method 2: Using get_doc with keyword arguments
    book2 = frappe.get_doc(
        doctype="Library Book",
        book_name="Data Science Handbook",
        author="Jane Smith",
        isbn="978-0-987654-32-1"
    )
    book2.insert()
    
    return book.name

def retrieve_book():
    """Retrieve and work with existing documents"""
    
    # Get single document
    book = frappe.get_doc("Library Book", "BOOK-001")
    
    # Get document for update (locks record)
    book_for_update = frappe.get_doc("Library Book", "BOOK-001", for_update=True)
    
    # Access fields
    print(f"Book: {book.book_name}")
    print(f"Author: {book.author}")
    print(f"Status: {book.status}")
    
    # Modify and save
    book.status = "Issued"
    book.save()
    
    # Reload from database
    book.reload()
    
    return book

def work_with_child_tables():
    """Work with documents having child tables"""
    
    # Create document with child table
    order = frappe.get_doc({
        "doctype": "Library Order",
        "order_date": frappe.utils.today(),
        "supplier": "Book Distributor Inc",
        "items": [
            {
                "book": "BOOK-001",
                "quantity": 5,
                "rate": 25.00
            },
            {
                "book": "BOOK-002", 
                "quantity": 3,
                "rate": 35.00
            }
        ]
    })
    order.insert()
    
    # Access child table rows
    for item in order.items:
        print(f"Book: {item.book}, Qty: {item.quantity}, Rate: {item.rate}")
        
    # Add new child row
    order.append("items", {
        "book": "BOOK-003",
        "quantity": 2,
        "rate": 45.00
    })
    
    # Remove child row
    order.remove(order.items[0])  # Remove first item
    
    # Update child row
    if order.items:
        order.items[0].quantity = 10
        
    order.save()
    
    return order.name
```

## Database Connection Management

### Connection Lifecycle

```python
def database_connection_management():
    """Understanding database connection management"""
    
    # Get current database connection
    db = frappe.db
    
    # Connection properties
    print(f"Database: {db.cur_db_name}")
    print(f"User: {db.user}")
    print(f"Host: {db.host}")
    
    # Transaction state
    print(f"In transaction: {db.in_transaction}")
    print(f"Transaction writes: {db.transaction_writes}")
    
    # Connection methods
    db.begin()     # Start transaction
    db.commit()    # Commit transaction
    db.rollback()  # Rollback transaction

def connection_callbacks():
    """Register callbacks for database events"""
    
    def before_commit():
        print("About to commit transaction")
        # Perform final validations
        # Clear caches
        
    def after_commit():
        print("Transaction committed successfully")
        # Send notifications
        # Update external systems
        
    # Register callbacks
    frappe.db.before_commit.add(before_commit)
    frappe.db.after_commit.add(after_commit)
    
    # Perform database operations
    book = frappe.get_doc({
        "doctype": "Library Book",
        "book_name": "Callback Example"
    })
    book.insert()
    
    # Callbacks will be triggered on commit
    frappe.db.commit()

@contextmanager
def temporary_db_connection():
    """Context manager for temporary database operations"""
    
    # Save current connection
    original_db = frappe.local.db
    
    try:
        # Create new connection
        frappe.connect()
        new_db = frappe.local.db
        
        # Use new connection
        yield new_db
        
    finally:
        # Restore original connection
        frappe.local.db = original_db
        
# Usage
with temporary_db_connection() as temp_db:
    # Operations using temporary connection
    temp_db.sql("SELECT COUNT(*) FROM `tabLibrary Book`")
```

## Query Building and Execution

### Basic Query Operations

```python
def basic_database_queries():
    """Basic database query operations"""
    
    # Get all documents
    books = frappe.get_all("Library Book")
    print(f"Total books: {len(books)}")
    
    # Get all with specific fields
    book_list = frappe.get_all("Library Book", 
        fields=["name", "book_name", "author", "status"]
    )
    
    # Get all with filters
    available_books = frappe.get_all("Library Book",
        filters={"status": "Available"},
        fields=["name", "book_name", "author"]
    )
    
    # Get all with multiple filters
    recent_programming_books = frappe.get_all("Library Book",
        filters={
            "category": "Programming",
            "publication_year": [">", 2020],
            "status": ["!=", "Archived"]
        },
        order_by="publication_year desc"
    )
    
    # Get all with OR conditions
    popular_books = frappe.get_all("Library Book",
        or_filters=[
            {"category": "Programming"},
            {"category": "Data Science"},
            {"rating": [">", 4.5]}
        ]
    )
    
    return book_list

def advanced_filtering():
    """Advanced filtering techniques"""
    
    # Complex filters using lists
    tech_books = frappe.get_all("Library Book",
        filters={
            "category": ["in", ["Programming", "Data Science", "AI/ML"]],
            "publication_year": ["between", [2018, 2023]],
            "rating": [">=", 4.0],
            "status": ["not in", ["Archived", "Damaged"]]
        }
    )
    
    # LIKE filters for text search
    python_books = frappe.get_all("Library Book",
        filters={
            "book_name": ["like", "%Python%"],
            "author": ["not like", "%Deprecated%"]
        }
    )
    
    # Date range filters
    from frappe.utils import add_days, today
    
    recent_issues = frappe.get_all("Library Issue",
        filters={
            "issue_date": [">=", add_days(today(), -30)],
            "return_date": ["is", "not set"]
        }
    )
    
    # Null/Not null filters
    books_without_isbn = frappe.get_all("Library Book",
        filters={"isbn": ["is", "not set"]}
    )
    
    books_with_isbn = frappe.get_all("Library Book",
        filters={"isbn": ["is", "set"]}
    )
    
    return tech_books

def pagination_and_sorting():
    """Pagination and sorting examples"""
    
    # Basic pagination
    page1_books = frappe.get_all("Library Book",
        fields=["name", "book_name", "author"],
        limit_start=0,
        limit_page_length=10,
        order_by="book_name asc"
    )
    
    page2_books = frappe.get_all("Library Book",
        fields=["name", "book_name", "author"],
        limit_start=10,
        limit_page_length=10,
        order_by="book_name asc"
    )
    
    # Multiple column sorting
    sorted_books = frappe.get_all("Library Book",
        order_by="category asc, publication_year desc, book_name asc"
    )
    
    # Random ordering
    random_books = frappe.get_all("Library Book",
        order_by="RAND()",
        limit_page_length=5
    )
    
    return page1_books
```

### Query Builder Usage

Based on analysis of `frappe/query_builder/builder.py:32-62`:

```python
def modern_query_builder():
    """Using Frappe's modern query builder"""
    
    from frappe.query_builder import DocType, Field
    from frappe.query_builder.functions import Count, Sum, Avg, Max, Min
    
    # Define tables
    LibraryBook = DocType("Library Book")
    LibraryIssue = DocType("Library Issue")
    LibraryMember = DocType("Library Member")
    
    # Basic select query
    books = (
        frappe.qb.from_(LibraryBook)
        .select(LibraryBook.name, LibraryBook.book_name, LibraryBook.author)
        .where(LibraryBook.status == "Available")
        .orderby(LibraryBook.book_name)
    ).run(as_dict=True)
    
    # Query with joins
    issued_books = (
        frappe.qb.from_(LibraryIssue)
        .inner_join(LibraryBook).on(LibraryIssue.book == LibraryBook.name)
        .inner_join(LibraryMember).on(LibraryIssue.member == LibraryMember.name)
        .select(
            LibraryBook.book_name,
            LibraryMember.full_name.as_("member_name"),
            LibraryIssue.issue_date,
            LibraryIssue.due_date
        )
        .where(LibraryIssue.return_date.isnull())
    ).run(as_dict=True)
    
    # Aggregation queries
    book_stats = (
        frappe.qb.from_(LibraryBook)
        .select(
            LibraryBook.category,
            Count("*").as_("total_books"),
            Count(LibraryBook.isbn).as_("books_with_isbn"),
            Avg(LibraryBook.rating).as_("avg_rating"),
            Max(LibraryBook.publication_year).as_("latest_year"),
            Min(LibraryBook.publication_year).as_("earliest_year")
        )
        .where(LibraryBook.status != "Archived")
        .groupby(LibraryBook.category)
        .having(Count("*") > 5)
        .orderby(Count("*"), order=frappe.qb.desc)
    ).run(as_dict=True)
    
    # Subqueries
    popular_categories = (
        frappe.qb.from_(LibraryBook)
        .select(LibraryBook.category)
        .where(LibraryBook.rating >= 4.0)
        .groupby(LibraryBook.category)
        .having(Count("*") > 10)
    )
    
    books_in_popular_categories = (
        frappe.qb.from_(LibraryBook)
        .select("*")
        .where(LibraryBook.category.isin(popular_categories))
    ).run(as_dict=True)
    
    # Complex conditions
    advanced_search = (
        frappe.qb.from_(LibraryBook)
        .select(LibraryBook.name, LibraryBook.book_name)
        .where(
            (LibraryBook.category == "Programming") |
            (
                (LibraryBook.rating >= 4.5) &
                (LibraryBook.publication_year >= 2020)
            )
        )
        .where(LibraryBook.status.notin(["Archived", "Damaged"]))
    ).run(as_dict=True)
    
    return books

def raw_sql_with_query_builder():
    """Combining raw SQL with query builder"""
    
    # Insert using query builder
    LibraryBook = DocType("Library Book")
    
    insert_query = (
        frappe.qb.into(LibraryBook)
        .insert(
            "Test Book QB",           # book_name
            "test-author@example.com", # author
            "978-0-qb-test-123",      # isbn
            2023,                     # publication_year
            "Available"               # status
        )
    )
    insert_query.run()
    
    # Update using query builder
    update_query = (
        frappe.qb.update(LibraryBook)
        .set(LibraryBook.status, "Issued")
        .where(LibraryBook.name == "BOOK-001")
    )
    update_query.run()
    
    # Delete using query builder
    delete_query = (
        frappe.qb.from_(LibraryBook)
        .delete()
        .where(LibraryBook.book_name == "Test Book QB")
    )
    delete_query.run()
    
    # Custom functions
    from frappe.query_builder.functions import Concat, Upper, Lower, Substring
    
    formatted_books = (
        frappe.qb.from_(LibraryBook)
        .select(
            Concat(Upper(Substring(LibraryBook.book_name, 1, 1)), 
                  Lower(Substring(LibraryBook.book_name, 2))).as_("formatted_name"),
            LibraryBook.author
        )
        .limit(10)
    ).run(as_dict=True)
    
    return formatted_books
```

## Schema Management

### Dynamic Schema Operations

Based on analysis of `frappe/database/schema.py:17-80`:

```python
class LibraryBookSchema:
    """Schema management for Library Book DocType"""
    
    def create_custom_fields(self):
        """Add custom fields to existing DocType"""
        
        # Add rating field
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Book",
            "label": "Rating",
            "fieldname": "rating",
            "fieldtype": "Rating",
            "insert_after": "publication_year",
            "description": "Book rating from 1 to 5 stars"
        }).insert()
        
        # Add category field with options
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Book", 
            "label": "Category",
            "fieldname": "category",
            "fieldtype": "Select",
            "options": "\nProgramming\nData Science\nAI/ML\nWeb Development\nMobile Development",
            "insert_after": "rating",
            "reqd": 1
        }).insert()
        
        # Add text editor field
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Book",
            "label": "Summary", 
            "fieldname": "summary",
            "fieldtype": "Text Editor",
            "insert_after": "category"
        }).insert()
        
    def create_child_table_field(self):
        """Add child table to DocType"""
        
        # First create the child DocType
        child_doctype = frappe.get_doc({
            "doctype": "DocType",
            "name": "Library Book Review",
            "module": "Library Management",
            "istable": 1,  # Mark as child table
            "autoname": "autoincrement",
            "fields": [
                {
                    "fieldname": "reviewer_name",
                    "label": "Reviewer Name", 
                    "fieldtype": "Data",
                    "reqd": 1
                },
                {
                    "fieldname": "review_rating",
                    "label": "Rating",
                    "fieldtype": "Rating", 
                    "reqd": 1
                },
                {
                    "fieldname": "review_text",
                    "label": "Review",
                    "fieldtype": "Text"
                },
                {
                    "fieldname": "review_date",
                    "label": "Review Date",
                    "fieldtype": "Date",
                    "default": "Today"
                }
            ]
        })
        child_doctype.insert()
        
        # Add child table field to parent DocType
        frappe.get_doc({
            "doctype": "Custom Field",
            "dt": "Library Book",
            "label": "Reviews",
            "fieldname": "reviews",
            "fieldtype": "Table",
            "options": "Library Book Review",
            "insert_after": "summary"
        }).insert()
        
    def add_database_indexes(self):
        """Add database indexes for performance"""
        
        # Add index on frequently queried fields
        frappe.db.add_index("Library Book", ["category", "status"])
        frappe.db.add_index("Library Book", ["publication_year"])
        frappe.db.add_index("Library Book", ["author"])
        
        # Add unique constraint
        frappe.db.add_unique("Library Book", ["isbn"])
        
        # Composite index for complex queries
        frappe.db.add_index("Library Book", ["category", "publication_year", "status"])

def schema_migration_example():
    """Example of schema migration operations"""
    
    def add_new_column():
        """Add new column to existing table"""
        
        if not frappe.db.has_column("Library Book", "digital_copy_url"):
            # Add column using ALTER TABLE
            frappe.db.sql("""
                ALTER TABLE `tabLibrary Book` 
                ADD COLUMN `digital_copy_url` VARCHAR(500) NULL
            """)
            
            # Update existing records
            frappe.db.sql("""
                UPDATE `tabLibrary Book` 
                SET `digital_copy_url` = NULL 
                WHERE `digital_copy_url` IS NULL
            """)
            
    def modify_column_type():
        """Change column data type"""
        
        # Change ISBN from VARCHAR(20) to VARCHAR(17) 
        frappe.db.sql("""
            ALTER TABLE `tabLibrary Book`
            MODIFY COLUMN `isbn` VARCHAR(17)
        """)
        
    def rename_column():
        """Rename existing column"""
        
        if frappe.db.has_column("Library Book", "book_title"):
            frappe.db.sql("""
                ALTER TABLE `tabLibrary Book`
                CHANGE COLUMN `book_title` `book_name` VARCHAR(255)
            """)
            
    def drop_column():
        """Remove unnecessary column"""
        
        if frappe.db.has_column("Library Book", "old_field"):
            frappe.db.sql("""
                ALTER TABLE `tabLibrary Book`
                DROP COLUMN `old_field`
            """)
            
    def add_constraints():
        """Add database constraints"""
        
        # Add CHECK constraint for publication year
        frappe.db.sql("""
            ALTER TABLE `tabLibrary Book`
            ADD CONSTRAINT `chk_publication_year` 
            CHECK (`publication_year` BETWEEN 1800 AND YEAR(NOW()) + 1)
        """)
        
        # Add foreign key constraint
        frappe.db.sql("""
            ALTER TABLE `tabLibrary Issue`
            ADD CONSTRAINT `fk_library_book`
            FOREIGN KEY (`book`) REFERENCES `tabLibrary Book`(`name`)
            ON DELETE RESTRICT ON UPDATE CASCADE
        """)
    
    # Execute migrations
    add_new_column()
    modify_column_type() 
    rename_column()
    add_constraints()
```

## Advanced Query Operations

### Complex Joins and Aggregations

```python
def complex_database_operations():
    """Advanced database operations with joins and aggregations"""
    
    # Complex join query with multiple tables
    library_stats = frappe.db.sql("""
        SELECT 
            b.category,
            COUNT(DISTINCT b.name) as total_books,
            COUNT(DISTINCT i.name) as total_issues,
            COUNT(DISTINCT m.name) as unique_borrowers,
            AVG(b.rating) as avg_rating,
            SUM(CASE WHEN i.return_date IS NULL THEN 1 ELSE 0 END) as currently_issued,
            AVG(DATEDIFF(COALESCE(i.return_date, NOW()), i.issue_date)) as avg_loan_duration
        FROM `tabLibrary Book` b
        LEFT JOIN `tabLibrary Issue` i ON b.name = i.book
        LEFT JOIN `tabLibrary Member` m ON i.member = m.name
        WHERE b.status != 'Archived'
        GROUP BY b.category
        HAVING total_books > 10
        ORDER BY total_issues DESC
    """, as_dict=True)
    
    # Window functions for ranking and analytics
    book_rankings = frappe.db.sql("""
        SELECT 
            book_name,
            category,
            rating,
            publication_year,
            ROW_NUMBER() OVER (PARTITION BY category ORDER BY rating DESC) as category_rank,
            RANK() OVER (ORDER BY rating DESC) as overall_rank,
            LAG(rating, 1) OVER (PARTITION BY category ORDER BY publication_year) as prev_year_rating,
            AVG(rating) OVER (PARTITION BY category) as category_avg_rating
        FROM `tabLibrary Book`
        WHERE status = 'Available' AND rating IS NOT NULL
    """, as_dict=True)
    
    # Recursive CTE for hierarchical data
    category_hierarchy = frappe.db.sql("""
        WITH RECURSIVE CategoryHierarchy AS (
            -- Base case: root categories
            SELECT name, parent_category, name as root_category, 0 as level
            FROM `tabBook Category` 
            WHERE parent_category IS NULL
            
            UNION ALL
            
            -- Recursive case: child categories
            SELECT c.name, c.parent_category, ch.root_category, ch.level + 1
            FROM `tabBook Category` c
            INNER JOIN CategoryHierarchy ch ON c.parent_category = ch.name
        )
        SELECT * FROM CategoryHierarchy ORDER BY root_category, level, name
    """, as_dict=True)
    
    return library_stats

def analytical_queries():
    """Analytical and reporting queries"""
    
    # Time series analysis
    monthly_circulation = frappe.db.sql("""
        SELECT 
            DATE_FORMAT(issue_date, '%Y-%m') as month,
            COUNT(*) as issues,
            COUNT(DISTINCT member) as unique_members,
            COUNT(DISTINCT book) as unique_books,
            AVG(DATEDIFF(COALESCE(return_date, NOW()), issue_date)) as avg_duration
        FROM `tabLibrary Issue`
        WHERE issue_date >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
        GROUP BY DATE_FORMAT(issue_date, '%Y-%m')
        ORDER BY month
    """, as_dict=True)
    
    # Cohort analysis
    member_cohorts = frappe.db.sql("""
        SELECT 
            DATE_FORMAT(m.creation, '%Y-%m') as signup_month,
            COUNT(DISTINCT m.name) as new_members,
            COUNT(DISTINCT i.member) as active_borrowers,
            COUNT(i.name) as total_issues,
            ROUND(COUNT(DISTINCT i.member) * 100.0 / COUNT(DISTINCT m.name), 2) as activation_rate
        FROM `tabLibrary Member` m
        LEFT JOIN `tabLibrary Issue` i ON m.name = i.member 
            AND i.issue_date BETWEEN m.creation AND DATE_ADD(m.creation, INTERVAL 30 DAY)
        WHERE m.creation >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
        GROUP BY DATE_FORMAT(m.creation, '%Y-%m')
        ORDER BY signup_month
    """, as_dict=True)
    
    # Advanced filtering with EXISTS
    books_never_issued = frappe.db.sql("""
        SELECT name, book_name, category, publication_year
        FROM `tabLibrary Book` b
        WHERE NOT EXISTS (
            SELECT 1 FROM `tabLibrary Issue` i 
            WHERE i.book = b.name
        )
        AND b.status = 'Available'
        ORDER BY b.creation DESC
    """, as_dict=True)
    
    # Performance analysis with execution plan
    frappe.db.sql("EXPLAIN SELECT * FROM `tabLibrary Book` WHERE category = 'Programming'")
    
    return monthly_circulation

def bulk_operations():
    """Efficient bulk database operations"""
    
    # Bulk insert with executemany
    book_data = [
        ("Bulk Book 1", "Author 1", "978-0-111-111-111", 2023, "Available"),
        ("Bulk Book 2", "Author 2", "978-0-222-222-222", 2023, "Available"),
        ("Bulk Book 3", "Author 3", "978-0-333-333-333", 2023, "Available"),
    ]
    
    frappe.db.sql("""
        INSERT INTO `tabLibrary Book` 
        (name, book_name, author, isbn, publication_year, status, creation, modified, owner, modified_by)
        VALUES 
    """ + ",".join([
        f"(UUID(), %s, %s, %s, %s, %s, NOW(), NOW(), %s, %s)"
        for _ in book_data
    ]), [item + (frappe.session.user, frappe.session.user) for item in book_data])
    
    # Bulk update using CASE statements
    rating_updates = [
        ("BOOK-001", 4.5),
        ("BOOK-002", 3.8),
        ("BOOK-003", 4.2),
    ]
    
    book_names = [update[0] for update in rating_updates]
    case_conditions = []
    
    for book_name, rating in rating_updates:
        case_conditions.append(f"WHEN '{book_name}' THEN {rating}")
    
    frappe.db.sql(f"""
        UPDATE `tabLibrary Book`
        SET rating = CASE name {' '.join(case_conditions)} ELSE rating END,
            modified = NOW(),
            modified_by = %s
        WHERE name IN ({','.join(['%s'] * len(book_names))})
    """, [frappe.session.user] + book_names)
    
    # Bulk delete with conditions
    frappe.db.sql("""
        DELETE FROM `tabLibrary Book`
        WHERE status = 'Archived' 
        AND creation < DATE_SUB(NOW(), INTERVAL 5 YEAR)
        AND NOT EXISTS (
            SELECT 1 FROM `tabLibrary Issue` 
            WHERE book = `tabLibrary Book`.name
        )
    """)
```

## Transaction Management

### Transaction Patterns

```python
def transaction_management():
    """Database transaction management patterns"""
    
    def basic_transaction():
        """Basic transaction with commit/rollback"""
        
        try:
            frappe.db.begin()
            
            # Create book
            book = frappe.get_doc({
                "doctype": "Library Book",
                "book_name": "Transaction Example",
                "author": "Test Author"
            })
            book.insert()
            
            # Create related issue
            issue = frappe.get_doc({
                "doctype": "Library Issue", 
                "member": "MEMBER-001",
                "book": book.name,
                "issue_date": frappe.utils.today()
            })
            issue.insert()
            
            # Both operations succeed - commit
            frappe.db.commit()
            
        except Exception as e:
            # Any operation fails - rollback
            frappe.db.rollback()
            frappe.throw(f"Transaction failed: {str(e)}")
            
    def nested_transactions():
        """Handle nested transactions with savepoints"""
        
        frappe.db.begin()
        
        try:
            # Outer transaction operations
            book = frappe.get_doc({
                "doctype": "Library Book", 
                "book_name": "Nested Transaction Book"
            })
            book.insert()
            
            # Create savepoint for nested operation
            savepoint_name = "book_issue_savepoint"
            frappe.db.savepoint(savepoint_name)
            
            try:
                # Inner transaction operations
                issue = frappe.get_doc({
                    "doctype": "Library Issue",
                    "member": "INVALID-MEMBER",  # This will fail
                    "book": book.name
                })
                issue.insert()
                
                # Release savepoint if successful
                frappe.db.release_savepoint(savepoint_name)
                
            except Exception as inner_e:
                # Rollback to savepoint only
                frappe.db.rollback_savepoint(savepoint_name)
                print(f"Inner operation failed: {inner_e}")
                # Outer transaction can still continue
                
            # Commit outer transaction
            frappe.db.commit()
            
        except Exception as outer_e:
            # Rollback entire transaction
            frappe.db.rollback()
            raise outer_e
            
    @frappe.db.sql_transaction
    def decorated_transaction():
        """Using transaction decorator"""
        
        # All operations in this function are wrapped in a transaction
        # Automatic rollback on any exception
        
        book = frappe.get_doc({
            "doctype": "Library Book",
            "book_name": "Decorated Transaction Book"
        })
        book.insert()
        
        member = frappe.get_doc({
            "doctype": "Library Member",
            "email": "decorated@example.com",
            "first_name": "Decorated"
        })
        member.insert()
        
        return book.name, member.name
        
    # Execute examples
    basic_transaction()
    nested_transactions()
    decorated_transaction()

@contextmanager
def atomic_operation():
    """Context manager for atomic operations"""
    
    original_auto_commit = frappe.db.auto_commit_on_many_writes
    frappe.db.auto_commit_on_many_writes = False
    
    frappe.db.begin()
    
    try:
        yield
        frappe.db.commit()
        
    except Exception:
        frappe.db.rollback()
        raise
        
    finally:
        frappe.db.auto_commit_on_many_writes = original_auto_commit

# Usage
def batch_book_processing():
    """Process multiple books atomically"""
    
    with atomic_operation():
        books_data = [
            {"book_name": "Book 1", "author": "Author 1"},
            {"book_name": "Book 2", "author": "Author 2"},
            {"book_name": "Book 3", "author": "Author 3"},
        ]
        
        for book_data in books_data:
            book = frappe.get_doc(dict(doctype="Library Book", **book_data))
            book.insert()
        
        # All books created successfully or none at all
```

## Performance Optimization

### Query Optimization

```python
def query_performance_optimization():
    """Database query performance optimization techniques"""
    
    def efficient_counting():
        """Efficient counting strategies"""
        
        # ❌ Slow - loads all records
        # all_books = frappe.get_all("Library Book")
        # count = len(all_books)
        
        # ✅ Fast - database-level counting
        count = frappe.db.count("Library Book")
        
        # ✅ Fast - counting with conditions
        available_count = frappe.db.count("Library Book", {"status": "Available"})
        
        # ✅ Fast - complex counting
        recent_count = frappe.db.sql("""
            SELECT COUNT(*) 
            FROM `tabLibrary Book` 
            WHERE publication_year >= 2020 AND status != 'Archived'
        """)[0][0]
        
        return count
        
    def efficient_existence_checks():
        """Efficient existence checking"""
        
        # ❌ Slow - retrieves full document
        # book = frappe.get_doc("Library Book", "BOOK-001")
        # exists = bool(book)
        
        # ✅ Fast - existence check only
        exists = frappe.db.exists("Library Book", "BOOK-001")
        
        # ✅ Fast - conditional existence
        isbn_exists = frappe.db.exists("Library Book", {"isbn": "978-0-123456-78-9"})
        
        return exists
        
    def pagination_optimization():
        """Optimized pagination for large datasets"""
        
        # ❌ Slow for large offsets - LIMIT 10000, 10
        # books = frappe.get_all("Library Book", 
        #     limit_start=10000, limit_page_length=10)
        
        # ✅ Fast - cursor-based pagination
        def get_books_after_cursor(cursor=None, limit=10):
            conditions = []
            values = []
            
            if cursor:
                conditions.append("name > %s")
                values.append(cursor)
                
            where_clause = "WHERE " + " AND ".join(conditions) if conditions else ""
            
            books = frappe.db.sql(f"""
                SELECT name, book_name, author, creation
                FROM `tabLibrary Book`
                {where_clause}
                ORDER BY name
                LIMIT %s
            """, values + [limit], as_dict=True)
            
            return books
            
        # Usage
        first_page = get_books_after_cursor(limit=10)
        if first_page:
            next_page = get_books_after_cursor(cursor=first_page[-1].name, limit=10)
            
    def batch_processing():
        """Process large datasets in batches"""
        
        def process_books_in_batches():
            """Process all books in manageable batches"""
            
            batch_size = 1000
            processed = 0
            
            while True:
                books = frappe.get_all("Library Book",
                    fields=["name", "book_name", "isbn"],
                    limit_start=processed,
                    limit_page_length=batch_size,
                    order_by="creation"
                )
                
                if not books:
                    break
                    
                # Process current batch
                for book in books:
                    # Perform operations on each book
                    validate_book_isbn(book)
                    
                processed += len(books)
                
                # Commit after each batch to avoid large transactions
                frappe.db.commit()
                
                print(f"Processed {processed} books...")
                
        def process_books_with_iterator():
            """Use database iterator for memory efficiency"""
            
            # Custom iterator for large datasets
            def book_iterator(batch_size=1000):
                offset = 0
                
                while True:
                    books = frappe.db.sql("""
                        SELECT name, book_name, isbn, status
                        FROM `tabLibrary Book`
                        ORDER BY creation
                        LIMIT %s OFFSET %s
                    """, (batch_size, offset), as_dict=True)
                    
                    if not books:
                        break
                        
                    for book in books:
                        yield book
                        
                    offset += batch_size
                    
            # Process books one by one without loading all into memory
            for book in book_iterator():
                # Process individual book
                if book.status == "Available":
                    update_book_availability(book)
                    
    # Execute optimization examples
    efficient_counting()
    efficient_existence_checks()
    pagination_optimization()
    batch_processing()

def caching_strategies():
    """Database caching strategies"""
    
    @frappe.cache.cache_result(ttl=300)  # Cache for 5 minutes
    def get_popular_categories():
        """Cache expensive aggregation queries"""
        
        return frappe.db.sql("""
            SELECT 
                category,
                COUNT(*) as book_count,
                AVG(rating) as avg_rating
            FROM `tabLibrary Book`
            WHERE status = 'Available'
            GROUP BY category
            HAVING book_count > 5
            ORDER BY avg_rating DESC
        """, as_dict=True)
        
    def manual_caching():
        """Manual caching with Redis"""
        
        cache_key = "library:book_stats"
        
        # Try to get from cache
        cached_stats = frappe.cache.get(cache_key)
        if cached_stats:
            return cached_stats
            
        # Calculate and cache
        stats = frappe.db.sql("""
            SELECT 
                COUNT(*) as total_books,
                COUNT(CASE WHEN status = 'Available' THEN 1 END) as available_books,
                COUNT(CASE WHEN status = 'Issued' THEN 1 END) as issued_books,
                AVG(rating) as avg_rating
            FROM `tabLibrary Book`
        """, as_dict=True)[0]
        
        # Cache for 10 minutes
        frappe.cache.setex(cache_key, 600, stats)
        
        return stats
        
    def cache_invalidation():
        """Cache invalidation patterns"""
        
        def clear_book_caches():
            """Clear all book-related caches"""
            
            cache_keys = [
                "library:book_stats",
                "library:popular_categories", 
                "library:available_books_count"
            ]
            
            for key in cache_keys:
                frappe.cache.delete(key)
                
        # Clear caches when books are modified
        def on_book_update(doc, method):
            clear_book_caches()
            
        # Register in hooks.py
        # doc_events = {
        #     "Library Book": {
        #         "on_update": "library_management.utils.on_book_update",
        #         "after_insert": "library_management.utils.on_book_update",
        #         "on_trash": "library_management.utils.on_book_update"
        #     }
        # }
        
    return get_popular_categories()
```

## Database Utilities

### Utility Functions

```python
def database_utilities():
    """Useful database utility functions"""
    
    def get_single_value():
        """Get single field value efficiently"""
        
        # Get single field from single document
        book_name = frappe.db.get_value("Library Book", "BOOK-001", "book_name")
        
        # Get multiple fields
        book_data = frappe.db.get_value("Library Book", "BOOK-001", 
            ["book_name", "author", "isbn"], as_dict=True)
        
        # Get with filters
        latest_book = frappe.db.get_value("Library Book",
            filters={"status": "Available"},
            fieldname="book_name",
            order_by="creation desc"
        )
        
        return book_data
        
    def set_single_value():
        """Set single field value efficiently"""
        
        # Set single field
        frappe.db.set_value("Library Book", "BOOK-001", "status", "Issued")
        
        # Set multiple fields
        frappe.db.set_value("Library Book", "BOOK-001", {
            "status": "Available",
            "last_issued_date": None,
            "current_borrower": None
        })
        
        # Set with update_modified
        frappe.db.set_value("Library Book", "BOOK-001", 
            "rating", 4.5, update_modified=True)
            
    def bulk_set_values():
        """Efficiently update multiple records"""
        
        # Update all books in a category
        frappe.db.sql("""
            UPDATE `tabLibrary Book`
            SET status = 'Under Review'
            WHERE category = 'Programming' AND publication_year < 2015
        """)
        
        # Bulk update with specific conditions
        book_updates = [
            ("BOOK-001", 4.5),
            ("BOOK-002", 3.8),
            ("BOOK-003", 4.2)
        ]
        
        for book_id, rating in book_updates:
            frappe.db.set_value("Library Book", book_id, "rating", rating)
            
    def database_introspection():
        """Inspect database structure and metadata"""
        
        # Get table columns
        columns = frappe.db.get_table_columns("Library Book")
        print("Table columns:", columns)
        
        # Check if column exists
        has_rating = frappe.db.has_column("Library Book", "rating")
        print(f"Has rating column: {has_rating}")
        
        # Get table indexes
        indexes = frappe.db.sql("""
            SHOW INDEX FROM `tabLibrary Book`
        """, as_dict=True)
        
        # Database size and statistics
        db_size = frappe.db.sql("""
            SELECT 
                table_name,
                table_rows,
                data_length,
                index_length,
                (data_length + index_length) as total_size
            FROM information_schema.tables
            WHERE table_schema = %s
            AND table_name LIKE 'tab%'
            ORDER BY total_size DESC
        """, (frappe.conf.db_name,), as_dict=True)
        
        return db_size
        
    def data_validation():
        """Database data validation utilities"""
        
        def find_orphaned_records():
            """Find records with broken references"""
            
            # Find library issues with non-existent books
            orphaned_issues = frappe.db.sql("""
                SELECT i.name, i.book
                FROM `tabLibrary Issue` i
                LEFT JOIN `tabLibrary Book` b ON i.book = b.name
                WHERE b.name IS NULL
            """, as_dict=True)
            
            # Find books with invalid authors
            invalid_authors = frappe.db.sql("""
                SELECT name, author
                FROM `tabLibrary Book`
                WHERE author IS NOT NULL
                AND author NOT IN (SELECT name FROM `tabUser`)
            """, as_dict=True)
            
            return orphaned_issues, invalid_authors
            
        def check_data_integrity():
            """Check data integrity constraints"""
            
            # Find duplicate ISBNs
            duplicate_isbns = frappe.db.sql("""
                SELECT isbn, COUNT(*) as count
                FROM `tabLibrary Book`
                WHERE isbn IS NOT NULL AND isbn != ''
                GROUP BY isbn
                HAVING count > 1
            """, as_dict=True)
            
            # Find invalid date ranges
            invalid_dates = frappe.db.sql("""
                SELECT name, issue_date, due_date, return_date
                FROM `tabLibrary Issue`
                WHERE due_date < issue_date
                OR (return_date IS NOT NULL AND return_date < issue_date)
            """, as_dict=True)
            
            return duplicate_isbns, invalid_dates
            
        def clean_orphaned_data():
            """Clean up orphaned and invalid data"""
            
            # Remove orphaned library issues
            frappe.db.sql("""
                DELETE i FROM `tabLibrary Issue` i
                LEFT JOIN `tabLibrary Book` b ON i.book = b.name
                WHERE b.name IS NULL
            """)
            
            # Remove empty or invalid records
            frappe.db.sql("""
                DELETE FROM `tabLibrary Book`
                WHERE book_name IS NULL OR book_name = ''
            """)
            
        return find_orphaned_records()
        
    # Execute utility examples
    get_single_value()
    set_single_value()
    database_introspection()
    data_validation()
```

## Raw SQL Operations

### Advanced SQL Techniques

```python
def raw_sql_operations():
    """Advanced raw SQL operations"""
    
    def dynamic_query_building():
        """Build dynamic queries safely"""
        
        def search_books(filters=None, sort_field=None, sort_order="asc"):
            """Dynamic book search with safe parameter binding"""
            
            base_query = """
                SELECT name, book_name, author, isbn, category, rating
                FROM `tabLibrary Book`
            """
            
            conditions = []
            values = []
            
            # Build WHERE conditions dynamically
            if filters:
                for field, value in filters.items():
                    if isinstance(value, list):
                        # Handle IN conditions
                        placeholders = ",".join(["%s"] * len(value))
                        conditions.append(f"`{field}` IN ({placeholders})")
                        values.extend(value)
                    elif isinstance(value, dict):
                        # Handle operator conditions
                        operator = value.get("operator", "=")
                        conditions.append(f"`{field}` {operator} %s")
                        values.append(value["value"])
                    else:
                        # Simple equality
                        conditions.append(f"`{field}` = %s")
                        values.append(value)
            
            # Add WHERE clause if conditions exist
            if conditions:
                base_query += " WHERE " + " AND ".join(conditions)
                
            # Add ORDER BY clause safely
            if sort_field:
                # Validate sort field to prevent SQL injection
                valid_fields = ["name", "book_name", "author", "creation", "rating"]
                if sort_field in valid_fields:
                    sort_order = "ASC" if sort_order.lower() == "asc" else "DESC"
                    base_query += f" ORDER BY `{sort_field}` {sort_order}"
                    
            return frappe.db.sql(base_query, values, as_dict=True)
            
        # Usage examples
        results1 = search_books({
            "category": ["Programming", "Data Science"],
            "rating": {"operator": ">=", "value": 4.0},
            "status": "Available"
        }, sort_field="rating", sort_order="desc")
        
        return results1
        
    def complex_analytical_queries():
        """Complex analytical SQL queries"""
        
        # Moving averages and window functions
        book_trends = frappe.db.sql("""
            SELECT 
                book_name,
                category,
                rating,
                publication_year,
                AVG(rating) OVER (
                    PARTITION BY category 
                    ORDER BY publication_year 
                    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
                ) as moving_avg_rating,
                ROW_NUMBER() OVER (
                    PARTITION BY category 
                    ORDER BY rating DESC
                ) as category_rank,
                FIRST_VALUE(book_name) OVER (
                    PARTITION BY category 
                    ORDER BY rating DESC
                    ROWS UNBOUNDED PRECEDING
                ) as top_book_in_category
            FROM `tabLibrary Book`
            WHERE rating IS NOT NULL
            ORDER BY category, rating DESC
        """, as_dict=True)
        
        # Pivot table simulation
        category_year_stats = frappe.db.sql("""
            SELECT 
                category,
                SUM(CASE WHEN publication_year >= 2020 THEN 1 ELSE 0 END) as books_2020_plus,
                SUM(CASE WHEN publication_year BETWEEN 2015 AND 2019 THEN 1 ELSE 0 END) as books_2015_2019,
                SUM(CASE WHEN publication_year < 2015 THEN 1 ELSE 0 END) as books_pre_2015,
                AVG(CASE WHEN publication_year >= 2020 THEN rating END) as avg_rating_recent,
                AVG(CASE WHEN publication_year < 2020 THEN rating END) as avg_rating_older
            FROM `tabLibrary Book`
            WHERE status != 'Archived'
            GROUP BY category
            ORDER BY category
        """, as_dict=True)
        
        # JSON aggregation (MySQL 5.7+)
        if frappe.conf.db_type == "mariadb":
            member_activity = frappe.db.sql("""
                SELECT 
                    m.name as member_id,
                    m.full_name,
                    COUNT(i.name) as total_issues,
                    JSON_ARRAYAGG(
                        JSON_OBJECT(
                            'book', b.book_name,
                            'issue_date', i.issue_date,
                            'return_date', i.return_date,
                            'rating', b.rating
                        )
                    ) as issue_history
                FROM `tabLibrary Member` m
                LEFT JOIN `tabLibrary Issue` i ON m.name = i.member
                LEFT JOIN `tabLibrary Book` b ON i.book = b.name
                GROUP BY m.name, m.full_name
                HAVING total_issues > 0
                ORDER BY total_issues DESC
            """, as_dict=True)
        
        return book_trends
        
    def stored_procedures_and_functions():
        """Working with stored procedures and functions"""
        
        def create_stored_procedure():
            """Create stored procedure for complex operations"""
            
            frappe.db.sql("""
                CREATE PROCEDURE IF NOT EXISTS CalculateLibraryStats(
                    IN category_filter VARCHAR(255),
                    OUT total_books INT,
                    OUT avg_rating DECIMAL(3,2),
                    OUT most_popular_book VARCHAR(255)
                )
                BEGIN
                    -- Calculate statistics
                    SELECT 
                        COUNT(*),
                        COALESCE(AVG(rating), 0),
                        (SELECT book_name FROM `tabLibrary Book` b2 
                         WHERE b2.category = category_filter 
                         ORDER BY b2.rating DESC LIMIT 1)
                    INTO total_books, avg_rating, most_popular_book
                    FROM `tabLibrary Book` b
                    WHERE b.category = category_filter;
                END
            """)
            
        def call_stored_procedure():
            """Call stored procedure and get results"""
            
            frappe.db.sql("""
                CALL CalculateLibraryStats('Programming', @total, @avg, @popular)
            """)
            
            result = frappe.db.sql("""
                SELECT @total as total_books, @avg as avg_rating, @popular as most_popular
            """, as_dict=True)[0]
            
            return result
            
        def create_custom_function():
            """Create custom database function"""
            
            frappe.db.sql("""
                CREATE FUNCTION IF NOT EXISTS CalculateLateFee(
                    issue_date DATE,
                    return_date DATE,
                    daily_rate DECIMAL(10,2)
                ) RETURNS DECIMAL(10,2)
                READS SQL DATA
                DETERMINISTIC
                BEGIN
                    DECLARE days_overdue INT DEFAULT 0;
                    
                    IF return_date > DATE_ADD(issue_date, INTERVAL 14 DAY) THEN
                        SET days_overdue = DATEDIFF(return_date, DATE_ADD(issue_date, INTERVAL 14 DAY));
                        RETURN days_overdue * daily_rate;
                    END IF;
                    
                    RETURN 0.00;
                END
            """)
            
        # Usage examples
        if frappe.db.db_type == "mariadb":
            create_stored_procedure()
            stats = call_stored_procedure()
            create_custom_function()
            
    def database_maintenance():
        """Database maintenance operations"""
        
        def analyze_and_optimize():
            """Analyze and optimize tables"""
            
            # Analyze table statistics
            frappe.db.sql("ANALYZE TABLE `tabLibrary Book`")
            frappe.db.sql("ANALYZE TABLE `tabLibrary Issue`")
            
            # Optimize tables (MySQL/MariaDB)
            frappe.db.sql("OPTIMIZE TABLE `tabLibrary Book`")
            
            # Check table integrity
            check_result = frappe.db.sql("CHECK TABLE `tabLibrary Book`", as_dict=True)
            print("Table check result:", check_result)
            
        def update_statistics():
            """Update database statistics"""
            
            if frappe.conf.db_type == "postgres":
                # PostgreSQL statistics update
                frappe.db.sql("ANALYZE")
            else:
                # MySQL/MariaDB statistics update  
                frappe.db.sql("ANALYZE TABLE `tabLibrary Book`")
                
        def rebuild_indexes():
            """Rebuild table indexes"""
            
            # Drop and recreate indexes if needed
            frappe.db.sql("ALTER TABLE `tabLibrary Book` DROP INDEX IF EXISTS idx_category_year")
            frappe.db.sql("""
                ALTER TABLE `tabLibrary Book` 
                ADD INDEX idx_category_year (category, publication_year)
            """)
            
        # Execute maintenance
        analyze_and_optimize()
        update_statistics()
        rebuild_indexes()
        
    # Execute raw SQL examples
    dynamic_query_building()
    complex_analytical_queries()
    database_maintenance()
```

## Best Practices

### Database Design Principles

```python
class DatabaseBestPractices:
    """Database design and usage best practices"""
    
    def naming_conventions(self):
        """Consistent naming conventions"""
        
        # ✅ Good naming patterns
        examples = {
            "tables": "tabDocType Name",  # Always prefixed with 'tab'
            "fields": "lowercase_with_underscores",
            "indexes": "idx_field_name or idx_composite_fields",
            "constraints": "fk_table_reference or chk_field_condition"
        }
        
        # ✅ Good field names
        good_field_names = [
            "book_name",           # Clear, descriptive
            "publication_year",    # Specific, unambiguous  
            "is_active",          # Boolean prefix
            "created_date",       # Date suffix
            "total_amount",       # Calculation result
        ]
        
        # ❌ Bad field names to avoid
        bad_field_names = [
            "name1",              # Generic, unclear
            "data",               # Too vague
            "temp",               # Temporary names
            "flag",               # Unclear boolean
        ]
        
    def field_type_selection(self):
        """Choose appropriate field types"""
        
        field_type_guide = {
            # Text fields
            "short_text": "Data (VARCHAR 140)",
            "long_text": "Text (TEXT)",
            "rich_text": "Text Editor (LONGTEXT)",
            
            # Numeric fields  
            "integer": "Int",
            "decimal": "Currency/Float",
            "percentage": "Percent",
            
            # Date/Time fields
            "date_only": "Date",
            "date_time": "Datetime", 
            "time_only": "Time",
            
            # Selection fields
            "predefined_options": "Select",
            "multiple_selection": "MultiSelectPills",
            
            # Reference fields
            "link_to_doctype": "Link",
            "dynamic_link": "Dynamic Link",
            "table_reference": "Table",
            
            # Special fields
            "file_attachment": "Attach",
            "image": "Attach Image", 
            "rating": "Rating",
            "json_data": "JSON",
        }
        
    def index_strategy(self):
        """Strategic index creation"""
        
        def create_performance_indexes():
            """Create indexes for common query patterns"""
            
            # Single column indexes for frequently filtered fields
            indexes = [
                "ALTER TABLE `tabLibrary Book` ADD INDEX idx_status (status)",
                "ALTER TABLE `tabLibrary Book` ADD INDEX idx_category (category)",
                "ALTER TABLE `tabLibrary Book` ADD INDEX idx_author (author)",
                "ALTER TABLE `tabLibrary Issue` ADD INDEX idx_issue_date (issue_date)",
            ]
            
            # Composite indexes for multi-column queries
            composite_indexes = [
                """ALTER TABLE `tabLibrary Book` 
                   ADD INDEX idx_category_status (category, status)""",
                """ALTER TABLE `tabLibrary Book` 
                   ADD INDEX idx_year_rating (publication_year, rating)""",
                """ALTER TABLE `tabLibrary Issue` 
                   ADD INDEX idx_member_status (member, return_date)""",
            ]
            
            # Unique indexes for data integrity
            unique_indexes = [
                "ALTER TABLE `tabLibrary Book` ADD UNIQUE INDEX uk_isbn (isbn)",
                "ALTER TABLE `tabLibrary Member` ADD UNIQUE INDEX uk_email (email)",
            ]
            
            for index_sql in indexes + composite_indexes + unique_indexes:
                try:
                    frappe.db.sql(index_sql)
                except Exception as e:
                    print(f"Index creation failed: {e}")
                    
        def index_maintenance():
            """Monitor and maintain indexes"""
            
            # Check index usage statistics
            index_stats = frappe.db.sql("""
                SELECT 
                    table_name,
                    index_name, 
                    column_name,
                    cardinality,
                    nullable
                FROM information_schema.statistics
                WHERE table_schema = %s
                AND table_name LIKE 'tab%'
                ORDER BY table_name, index_name
            """, (frappe.conf.db_name,), as_dict=True)
            
            # Identify unused indexes (requires performance schema)
            unused_indexes = frappe.db.sql("""
                SELECT 
                    object_schema,
                    object_name,
                    index_name
                FROM performance_schema.table_io_waits_summary_by_index_usage
                WHERE index_name IS NOT NULL
                AND index_name != 'PRIMARY'
                AND count_read = 0
                AND count_write = 0
                AND count_fetch = 0
                AND object_schema = %s
            """, (frappe.conf.db_name,), as_dict=True)
            
            return index_stats
            
    def query_optimization(self):
        """Query optimization techniques"""
        
        def efficient_query_patterns():
            """Examples of efficient vs inefficient queries"""
            
            # ✅ Efficient patterns
            efficient_queries = {
                "use_indexes": """
                    SELECT name, book_name FROM `tabLibrary Book`
                    WHERE category = 'Programming'  -- Uses idx_category
                    AND status = 'Available'        -- Uses idx_category_status
                """,
                
                "limit_results": """
                    SELECT name, book_name FROM `tabLibrary Book`
                    ORDER BY rating DESC
                    LIMIT 10  -- Don't fetch more than needed
                """,
                
                "specific_fields": """
                    SELECT name, book_name, author  -- Only needed fields
                    FROM `tabLibrary Book`
                    WHERE status = 'Available'
                """,
                
                "join_optimization": """
                    SELECT b.book_name, m.full_name
                    FROM `tabLibrary Issue` i
                    INNER JOIN `tabLibrary Book` b ON i.book = b.name
                    INNER JOIN `tabLibrary Member` m ON i.member = m.name
                    WHERE i.return_date IS NULL  -- Filter early
                """,
            }
            
            # ❌ Inefficient patterns to avoid
            inefficient_queries = {
                "no_indexes": """
                    SELECT * FROM `tabLibrary Book`
                    WHERE UPPER(book_name) LIKE '%PYTHON%'  -- Function prevents index use
                """,
                
                "unnecessary_fields": """
                    SELECT * FROM `tabLibrary Book`  -- Fetches all columns
                    WHERE status = 'Available'
                """,
                
                "cartesian_joins": """
                    SELECT b.book_name, m.full_name
                    FROM `tabLibrary Book` b, `tabLibrary Member` m  -- No join condition
                """,
                
                "subquery_in_select": """
                    SELECT 
                        book_name,
                        (SELECT COUNT(*) FROM `tabLibrary Issue` 
                         WHERE book = `tabLibrary Book`.name) as issue_count  -- Slow subquery
                    FROM `tabLibrary Book`
                """
            }
            
            return efficient_queries
            
        def query_analysis():
            """Analyze query performance"""
            
            def explain_query(query):
                """Get query execution plan"""
                
                explain_result = frappe.db.sql(f"EXPLAIN {query}", as_dict=True)
                
                performance_indicators = []
                for row in explain_result:
                    if row.get('type') == 'ALL':
                        performance_indicators.append("Full table scan detected")
                    if row.get('Extra', '').find('Using filesort') != -1:
                        performance_indicators.append("Filesort required")
                    if row.get('Extra', '').find('Using temporary') != -1:
                        performance_indicators.append("Temporary table used")
                        
                return explain_result, performance_indicators
                
            # Example query analysis
            sample_query = """
                SELECT b.book_name, COUNT(i.name) as issue_count
                FROM `tabLibrary Book` b
                LEFT JOIN `tabLibrary Issue` i ON b.name = i.book
                GROUP BY b.name, b.book_name
                ORDER BY issue_count DESC
            """
            
            plan, issues = explain_query(sample_query)
            return plan, issues
            
    def connection_management(self):
        """Database connection best practices"""
        
        def connection_pooling():
            """Efficient connection usage"""
            
            # ✅ Use connection pooling (configured in site_config.json)
            connection_config = {
                "db_pool_size": 10,           # Maximum connections
                "db_timeout": 30,             # Connection timeout
                "db_max_overflow": 20,        # Additional connections when needed
                "db_pre_ping": True,          # Verify connections before use
            }
            
        def transaction_guidelines():
            """Transaction management guidelines"""
            
            # ✅ Keep transactions short
            def short_transaction():
                frappe.db.begin()
                try:
                    # Quick operations only
                    book = frappe.get_doc("Library Book", "BOOK-001")
                    book.status = "Issued"
                    book.save()
                    frappe.db.commit()
                except:
                    frappe.db.rollback()
                    raise
                    
            # ✅ Batch related operations
            def batched_operations():
                frappe.db.begin()
                try:
                    # Process related changes together
                    for book_id in ["BOOK-001", "BOOK-002", "BOOK-003"]:
                        book = frappe.get_doc("Library Book", book_id)
                        book.status = "Under Review"
                        book.save()
                    frappe.db.commit()
                except:
                    frappe.db.rollback()
                    raise
                    
            # ❌ Avoid long-running transactions
            def avoid_long_transaction():
                # Don't do this:
                # frappe.db.begin()
                # for i in range(10000):  # Long-running loop
                #     process_book(i)
                # frappe.db.commit()
                
                # Do this instead:
                batch_size = 100
                for i in range(0, 10000, batch_size):
                    frappe.db.begin()
                    try:
                        for j in range(i, min(i + batch_size, 10000)):
                            process_book(j)
                        frappe.db.commit()
                    except:
                        frappe.db.rollback()
                        raise
                        
    def security_practices(self):
        """Database security best practices"""
        
        def sql_injection_prevention():
            """Prevent SQL injection attacks"""
            
            # ✅ Always use parameterized queries
            def safe_query(book_name):
                return frappe.db.sql("""
                    SELECT name, author FROM `tabLibrary Book`
                    WHERE book_name = %s
                """, (book_name,), as_dict=True)
                
            # ✅ Validate input before using in queries
            def search_books(search_term):
                if not search_term or len(search_term) < 2:
                    return []
                    
                # Sanitize search term
                search_term = frappe.utils.strip_html(search_term)
                search_term = search_term.replace('%', '\\%').replace('_', '\\_')
                
                return frappe.db.sql("""
                    SELECT name, book_name FROM `tabLibrary Book`
                    WHERE book_name LIKE %s
                """, (f"%{search_term}%",), as_dict=True)
                
            # ❌ Never use string formatting in SQL
            def dangerous_query(book_name):
                # DON'T DO THIS!
                # return frappe.db.sql(f"""
                #     SELECT * FROM `tabLibrary Book`
                #     WHERE book_name = '{book_name}'
                # """)
                pass
                
        def access_control():
            """Implement proper access control"""
            
            # Use DocPerm for document-level security
            # Use User Permissions for row-level security
            # Use Custom Permission methods for complex logic
            
            def check_book_access(book_name, user):
                """Check if user can access specific book"""
                
                if not frappe.has_permission("Library Book", "read", book_name, user):
                    frappe.throw("Access denied")
                    
                return frappe.get_doc("Library Book", book_name)

# Usage example
best_practices = DatabaseBestPractices()
best_practices.naming_conventions()
best_practices.index_strategy()
best_practices.query_optimization()