# Testing Framework - Complete Guide

> **Comprehensive reference for unit testing, integration testing, test data management, and automated testing workflows**

## Table of Contents

- [Testing Architecture Overview](#testing-architecture-overview)
- [Unit Testing Setup](#unit-testing-setup)
- [Test Base Classes](#test-base-classes)
- [Document Testing Patterns](#document-testing-patterns)
- [API Testing Strategies](#api-testing-strategies)
- [Database Testing](#database-testing)
- [Performance Testing](#performance-testing)
- [Test Data Management](#test-data-management)
- [Mocking and Isolation](#mocking-and-isolation)
- [Integration Testing](#integration-testing)
- [Automated Testing Workflows](#automated-testing-workflows)
- [Debugging Techniques](#debugging-techniques)
- [Best Practices](#best-practices)

## Testing Architecture Overview

### Framework Components

Based on analysis of `frappe/tests/utils.py:20-48`, Frappe provides a comprehensive testing framework:

```python
# Core testing components
class FrappeTestCase(unittest.TestCase):
    """Base test class for all Frappe tests"""
    
    TEST_SITE = "test_site"
    SHOW_TRANSACTION_COMMIT_WARNINGS = False
    maxDiff = 10_000  # Long diffs for detailed comparisons
    
    @classmethod
    def setUpClass(cls):
        # Database connection management
        cls._primary_connection = frappe.local.db
        cls._secondary_connection = None
        
        # Transaction management 
        frappe.db.commit()
        
        # Cleanup registration
        cls.addClassCleanup(_rollback_db)
```

### Test Discovery and Execution

```bash
# Run all tests
bench --site sitename run-tests

# Run specific app tests
bench --site sitename run-tests --app your_app

# Run specific test module
bench --site sitename run-tests --module tests.test_custom_module

# Run with coverage
bench --site sitename run-tests --coverage

# Run tests with verbose output
bench --site sitename run-tests --verbose
```

### Test File Organization

```
your_app/
├── your_app/
│   ├── doctype/
│   │   └── custom_doctype/
│   │       ├── custom_doctype.py
│   │       └── test_custom_doctype.py
│   └── tests/
│       ├── __init__.py
│       ├── test_utils.py
│       ├── test_api.py
│       └── test_integration.py
└── fixtures/
    └── custom_records.json
```

## Unit Testing Setup

### Basic Test Structure

```python
import frappe
from frappe.tests.utils import FrappeTestCase

class TestLibraryBook(FrappeTestCase):
    """Test cases for Library Book DocType"""
    
    @classmethod
    def setUpClass(cls):
        """Set up test data once for all test methods"""
        super().setUpClass()
        
        # Create test author
        if not frappe.db.exists("Author", "test-author@example.com"):
            author = frappe.get_doc({
                "doctype": "Author",
                "email": "test-author@example.com",
                "first_name": "Test",
                "last_name": "Author"
            })
            author.insert(ignore_permissions=True)
    
    def setUp(self):
        """Set up before each test method"""
        self.test_book_data = {
            "doctype": "Library Book",
            "book_name": "Test Programming Book",
            "author": "test-author@example.com",
            "isbn": "978-0-123456-78-9",
            "publication_year": 2023,
            "status": "Available"
        }
    
    def tearDown(self):
        """Clean up after each test method"""
        # Remove any test documents created
        frappe.db.delete("Library Book", {"book_name": "Test Programming Book"})
        
    def test_book_creation(self):
        """Test basic book document creation"""
        book = frappe.get_doc(self.test_book_data)
        book.insert()
        
        # Verify document was created
        self.assertTrue(frappe.db.exists("Library Book", book.name))
        
        # Verify field values
        saved_book = frappe.get_doc("Library Book", book.name)
        self.assertEqual(saved_book.book_name, "Test Programming Book")
        self.assertEqual(saved_book.status, "Available")
        
    def test_book_validation(self):
        """Test document validation rules"""
        # Test missing required field
        invalid_data = self.test_book_data.copy()
        del invalid_data["author"]
        
        with self.assertRaises(frappe.exceptions.MandatoryError):
            book = frappe.get_doc(invalid_data)
            book.insert()
            
        # Test invalid ISBN format
        invalid_data = self.test_book_data.copy()
        invalid_data["isbn"] = "invalid-isbn"
        
        with self.assertRaises(frappe.exceptions.ValidationError):
            book = frappe.get_doc(invalid_data)
            book.insert()
```

### Custom Assertions

Based on `frappe/tests/utils.py:50-83`:

```python
class TestDocumentOperations(FrappeTestCase):
    def test_document_comparison(self):
        """Test custom document assertion methods"""
        
        # Create expected document structure
        expected_book = {
            "book_name": "Python Guide",
            "isbn": "978-0-123456-78-9",
            "status": "Available",
            "tags": ["programming", "python"]
        }
        
        # Create actual document
        book = frappe.get_doc({
            "doctype": "Library Book",
            **expected_book
        })
        book.insert()
        
        # Use custom assertion for document comparison
        self.assertDocumentEqual(expected_book, book)
        
    def test_sequence_subset(self):
        """Test sequence subset assertion"""
        all_books = frappe.get_all("Library Book", pluck="name")
        available_books = frappe.get_all("Library Book", 
            filters={"status": "Available"}, pluck="name")
        
        # Assert available books are subset of all books
        self.assertSequenceSubset(all_books, available_books)
        
    def test_html_comparison(self):
        """Test HTML normalization for consistent comparison"""
        html1 = "<div><p>Hello World</p></div>"
        html2 = "<div>\n  <p>Hello World</p>\n</div>"
        
        # Normalize and compare HTML
        normalized1 = self.normalize_html(html1)
        normalized2 = self.normalize_html(html2)
        self.assertEqual(normalized1, normalized2)
```

## Test Base Classes

### FrappeTestCase Features

Based on analysis of `frappe/tests/utils.py:20-140`:

```python
class LibraryTestCase(FrappeTestCase):
    """Extended base class with library-specific utilities"""
    
    def create_test_book(self, **kwargs):
        """Utility method to create test books"""
        default_data = {
            "doctype": "Library Book",
            "book_name": frappe.generate_hash(),
            "author": "test-author@example.com",
            "isbn": f"978-0-{frappe.generate_hash()[:10]}",
            "status": "Available"
        }
        default_data.update(kwargs)
        
        book = frappe.get_doc(default_data)
        book.insert(ignore_permissions=True)
        return book
        
    def create_test_member(self, **kwargs):
        """Utility method to create test library members"""
        default_data = {
            "doctype": "Library Member",
            "first_name": frappe.mock("first_name"),
            "last_name": frappe.mock("last_name"),
            "email": frappe.mock("email"),
            "membership_type": "Basic"
        }
        default_data.update(kwargs)
        
        member = frappe.get_doc(default_data)
        member.insert(ignore_permissions=True)
        return member
        
    @contextmanager
    def temp_flag(self, flag_name, flag_value):
        """Temporarily set a system flag for testing"""
        original_value = frappe.flags.get(flag_name)
        try:
            frappe.flags[flag_name] = flag_value
            yield
        finally:
            if original_value is not None:
                frappe.flags[flag_name] = original_value
            else:
                frappe.flags.pop(flag_name, None)
```

### Database Connection Management

```python
class TestConcurrentOperations(FrappeTestCase):
    def test_multiple_connections(self):
        """Test operations using multiple database connections"""
        
        # Create document using primary connection
        with self.primary_connection():
            book = self.create_test_book()
            book_name = book.name
            
        # Verify from secondary connection
        with self.secondary_connection():
            # This simulates another user/process
            frappe.connect()
            exists = frappe.db.exists("Library Book", book_name)
            self.assertTrue(exists)
            
            # Modify document from secondary connection
            secondary_book = frappe.get_doc("Library Book", book_name)
            secondary_book.status = "Issued"
            secondary_book.save()
            
        # Verify change from primary connection
        with self.primary_connection():
            primary_book = frappe.get_doc("Library Book", book_name)
            self.assertEqual(primary_book.status, "Issued")
```

## Document Testing Patterns

### Lifecycle Testing

```python
class TestBookLifecycle(FrappeTestCase):
    def test_complete_book_workflow(self):
        """Test complete book lifecycle from creation to deletion"""
        
        # 1. Creation
        book = self.create_test_book()
        self.assertEqual(book.status, "Available")
        
        # 2. Issue book to member
        member = self.create_test_member()
        issue = frappe.get_doc({
            "doctype": "Library Issue",
            "member": member.name,
            "book": book.name,
            "issue_date": frappe.utils.today()
        })
        issue.insert()
        
        # Verify book status changed
        book.reload()
        self.assertEqual(book.status, "Issued")
        
        # 3. Return book
        issue.return_date = frappe.utils.add_days(frappe.utils.today(), 7)
        issue.save()
        
        # Verify book status changed back
        book.reload()
        self.assertEqual(book.status, "Available")
        
        # 4. Archive book
        book.status = "Archived"
        book.save()
        
        # Verify archived books cannot be issued
        with self.assertRaises(frappe.exceptions.ValidationError):
            new_issue = frappe.get_doc({
                "doctype": "Library Issue",
                "member": member.name,
                "book": book.name,
                "issue_date": frappe.utils.today()
            })
            new_issue.insert()
```

### Validation Testing

```python
class TestBookValidations(FrappeTestCase):
    def test_isbn_validation(self):
        """Test ISBN format validation"""
        
        # Valid ISBN-13
        valid_isbn = "978-0-123456-78-9"
        book = self.create_test_book(isbn=valid_isbn)
        self.assertEqual(book.isbn, valid_isbn)
        
        # Invalid ISBN format
        invalid_isbns = [
            "123-456-789",      # Too short
            "978-0-123456-789", # Invalid check digit
            "abc-def-ghi-jkl",  # Non-numeric
            ""                  # Empty
        ]
        
        for invalid_isbn in invalid_isbns:
            with self.assertRaises(frappe.exceptions.ValidationError):
                self.create_test_book(isbn=invalid_isbn)
                
    def test_publication_year_validation(self):
        """Test publication year validation"""
        
        current_year = frappe.utils.now_datetime().year
        
        # Valid years
        valid_years = [1900, 2000, current_year]
        for year in valid_years:
            book = self.create_test_book(publication_year=year)
            self.assertEqual(book.publication_year, year)
            
        # Invalid years
        invalid_years = [1800, current_year + 10]  # Too old or future
        for year in invalid_years:
            with self.assertRaises(frappe.exceptions.ValidationError):
                self.create_test_book(publication_year=year)
```

### Permission Testing

```python
class TestBookPermissions(FrappeTestCase):
    def test_role_based_access(self):
        """Test role-based document access"""
        
        # Create test users with different roles
        librarian = frappe.get_doc({
            "doctype": "User",
            "email": "librarian@example.com",
            "first_name": "Test",
            "last_name": "Librarian",
            "roles": [{"role": "Librarian"}]
        })
        librarian.insert(ignore_permissions=True)
        
        member_user = frappe.get_doc({
            "doctype": "User", 
            "email": "member@example.com",
            "first_name": "Test",
            "last_name": "Member",
            "roles": [{"role": "Library Member"}]
        })
        member_user.insert(ignore_permissions=True)
        
        # Test librarian can create books
        frappe.set_user("librarian@example.com")
        book = self.create_test_book()
        self.assertTrue(frappe.db.exists("Library Book", book.name))
        
        # Test member cannot create books
        frappe.set_user("member@example.com")
        with self.assertRaises(frappe.exceptions.PermissionError):
            self.create_test_book()
            
        # Test member can read books
        readable = frappe.has_permission("Library Book", "read")
        self.assertTrue(readable)
        
        # Reset to admin
        frappe.set_user("Administrator")
```

## API Testing Strategies

### REST API Testing

Based on analysis of `frappe/tests/test_api_v2.py:20-50`:

```python
from frappe.tests.test_api import FrappeAPITestCase

class TestLibraryAPI(FrappeAPITestCase):
    """Test REST API endpoints for library management"""
    
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Create test data
        cls.test_books = []
        for i in range(5):
            book = frappe.get_doc({
                "doctype": "Library Book",
                "book_name": f"Test Book {i}",
                "author": "test-author@example.com",
                "isbn": f"978-0-{i:09d}",
                "status": "Available"
            }).insert()
            cls.test_books.append(book.name)
        frappe.db.commit()
        
    def test_unauthorized_access(self):
        """Test API access without authentication"""
        import requests
        
        response = requests.get(self.resource("Library Book"))
        self.assertEqual(response.status_code, 403)
        
    def test_get_book_list(self):
        """Test GET /api/resource/Library%20Book"""
        
        response = self.get(self.resource("Library Book"), {"sid": self.sid})
        
        self.assertEqual(response.status_code, 200)
        self.assertIn("data", response.json)
        
        books = response.json["data"]
        self.assertGreaterEqual(len(books), 5)
        
        # Verify book structure
        book = books[0]
        required_fields = ["name", "book_name", "author", "status"]
        for field in required_fields:
            self.assertIn(field, book)
            
    def test_get_single_book(self):
        """Test GET /api/resource/Library%20Book/{name}"""
        
        book_name = self.test_books[0]
        response = self.get(self.resource("Library Book", book_name), {"sid": self.sid})
        
        self.assertEqual(response.status_code, 200)
        self.assertIn("data", response.json)
        
        book_data = response.json["data"]
        self.assertEqual(book_data["name"], book_name)
        
    def test_create_book_via_api(self):
        """Test POST /api/resource/Library%20Book"""
        
        new_book_data = {
            "book_name": "API Test Book",
            "author": "test-author@example.com", 
            "isbn": "978-0-999999999",
            "status": "Available"
        }
        
        response = self.post(
            self.resource("Library Book"),
            data=new_book_data,
            content_type="application/json",
            headers={"Authorization": f"Bearer {self.sid}"}
        )
        
        self.assertEqual(response.status_code, 200)
        
        # Verify book was created
        created_book = response.json["data"]
        self.assertEqual(created_book["book_name"], "API Test Book")
        
        # Clean up
        frappe.delete_doc("Library Book", created_book["name"])
        
    def test_api_filtering(self):
        """Test API with filters and sorting"""
        
        params = {
            "sid": self.sid,
            "filters": '{"status": "Available"}',
            "fields": '["name", "book_name", "status"]',
            "order_by": "book_name asc",
            "limit_start": 0,
            "limit_page_length": 3
        }
        
        response = self.get(self.resource("Library Book"), params)
        
        self.assertEqual(response.status_code, 200)
        books = response.json["data"]
        
        # Verify filtering
        for book in books:
            self.assertEqual(book["status"], "Available")
            
        # Verify field selection
        for book in books:
            self.assertIn("book_name", book)
            self.assertNotIn("author", book)  # Not in selected fields
            
        # Verify pagination
        self.assertLessEqual(len(books), 3)
```

### Webhook Testing

```python
class TestLibraryWebhooks(FrappeTestCase):
    def test_book_creation_webhook(self):
        """Test webhook triggered on book creation"""
        
        with patch('frappe.enqueue') as mock_enqueue:
            # Create book that should trigger webhook
            book = self.create_test_book()
            
            # Verify webhook was enqueued
            mock_enqueue.assert_called()
            
            # Get the webhook call
            webhook_calls = [call for call in mock_enqueue.call_args_list 
                           if "webhook" in str(call)]
            self.assertTrue(len(webhook_calls) > 0)
            
    def test_external_api_integration(self):
        """Test integration with external book database API"""
        
        with patch('requests.get') as mock_get:
            # Mock external API response
            mock_response = {
                "isbn": "978-0-123456-78-9",
                "title": "Advanced Python",
                "authors": ["John Doe"],
                "publisher": "Tech Books",
                "year": 2023
            }
            mock_get.return_value.json.return_value = mock_response
            mock_get.return_value.status_code = 200
            
            # Test book enrichment from external API
            book = frappe.get_doc({
                "doctype": "Library Book",
                "isbn": "978-0-123456-78-9"
            })
            
            # Trigger external API call
            book.enrich_from_external_api()
            
            # Verify data was populated
            self.assertEqual(book.book_name, "Advanced Python")
            self.assertEqual(book.publication_year, 2023)
            
            # Verify API was called correctly
            mock_get.assert_called_with(
                f"https://api.books.com/isbn/978-0-123456-78-9",
                timeout=30
            )
```

## Database Testing

### Query Testing

Based on analysis of `frappe/tests/test_db.py:23-50`:

```python
class TestLibraryQueries(FrappeTestCase):
    def test_database_operations(self):
        """Test various database operations"""
        
        # Test basic CRUD operations
        book_data = {
            "doctype": "Library Book",
            "book_name": "Database Test Book",
            "author": "test-author@example.com",
            "isbn": "978-0-db-test-123"
        }
        
        # Create
        book = frappe.get_doc(book_data)
        book.insert()
        book_name = book.name
        
        # Read
        fetched_book = frappe.get_doc("Library Book", book_name)
        self.assertEqual(fetched_book.book_name, "Database Test Book")
        
        # Update
        fetched_book.status = "Issued"
        fetched_book.save()
        
        # Verify update
        updated_book = frappe.get_value("Library Book", book_name, "status")
        self.assertEqual(updated_book, "Issued")
        
        # Delete
        frappe.delete_doc("Library Book", book_name)
        self.assertFalse(frappe.db.exists("Library Book", book_name))
        
    def test_complex_queries(self):
        """Test complex database queries"""
        
        # Create test data
        books = []
        for i in range(10):
            book = self.create_test_book(
                publication_year=2020 + (i % 3),
                status=["Available", "Issued", "Reserved"][i % 3]
            )
            books.append(book.name)
            
        # Test aggregation query
        stats = frappe.db.sql("""
            SELECT 
                status,
                COUNT(*) as count,
                AVG(publication_year) as avg_year
            FROM `tabLibrary Book` 
            WHERE name IN %(book_names)s
            GROUP BY status
            ORDER BY count DESC
        """, {"book_names": books}, as_dict=True)
        
        self.assertEqual(len(stats), 3)  # Three different statuses
        
        # Test query builder usage
        from frappe.query_builder import DocType
        LibraryBook = DocType("Library Book")
        
        recent_books = (
            frappe.qb.from_(LibraryBook)
            .select(LibraryBook.name, LibraryBook.book_name, LibraryBook.publication_year)
            .where(LibraryBook.publication_year >= 2022)
            .where(LibraryBook.status == "Available")
            .orderby(LibraryBook.publication_year, order=frappe.qb.desc)
        ).run(as_dict=True)
        
        for book in recent_books:
            self.assertGreaterEqual(book.publication_year, 2022)
            
    def test_transaction_handling(self):
        """Test database transaction handling"""
        
        initial_count = frappe.db.count("Library Book")
        
        try:
            # Start transaction
            frappe.db.begin()
            
            # Create multiple books
            for i in range(3):
                book = self.create_test_book(book_name=f"Transaction Test {i}")
                
            # Intentionally cause an error
            invalid_book = frappe.get_doc({
                "doctype": "Library Book", 
                "book_name": "Invalid Book",
                "author": "nonexistent-author"  # This should cause error
            })
            invalid_book.insert()
            
            frappe.db.commit()
            
        except Exception:
            # Rollback on error
            frappe.db.rollback()
            
        # Verify rollback worked - no new books created
        final_count = frappe.db.count("Library Book")
        self.assertEqual(initial_count, final_count)
```

## Performance Testing

### Query Performance

Based on analysis of `frappe/tests/test_perf.py:39-50`:

```python
class TestLibraryPerformance(FrappeTestCase):
    def setUp(self):
        super().setUp()
        # Create performance test data
        self.create_bulk_test_data()
        
    def create_bulk_test_data(self):
        """Create bulk data for performance testing"""
        
        # Create 100 books for performance testing
        books = []
        for i in range(100):
            book_data = {
                "doctype": "Library Book",
                "book_name": f"Perf Test Book {i:03d}",
                "author": f"author-{i % 10}@example.com",
                "isbn": f"978-0-{i:010d}",
                "publication_year": 2015 + (i % 8),
                "status": ["Available", "Issued", "Reserved"][i % 3]
            }
            books.append(book_data)
            
        # Bulk insert for better performance
        frappe.db.bulk_insert("Library Book", books)
        frappe.db.commit()
        
    def test_query_count_optimization(self):
        """Test that operations don't execute unnecessary queries"""
        
        # Warm up caches
        frappe.get_meta("Library Book")
        
        with self.assertQueryCount(1):
            # This should execute only one query
            books = frappe.get_all("Library Book", 
                fields=["name", "book_name", "status"],
                limit=10
            )
            
    def test_bulk_operations_performance(self):
        """Test performance of bulk operations"""
        
        import time
        
        # Test individual updates (slow)
        start_time = time.time()
        books = frappe.get_all("Library Book", limit=50, pluck="name")
        
        for book_name in books[:10]:  # Test with smaller subset
            frappe.db.set_value("Library Book", book_name, "remarks", "Updated individually")
            
        individual_time = time.time() - start_time
        
        # Test bulk update (fast)
        start_time = time.time()
        frappe.db.sql("""
            UPDATE `tabLibrary Book` 
            SET remarks = 'Updated in bulk'
            WHERE name IN %(names)s
        """, {"names": books[10:20]})  # Test with different subset
        
        bulk_time = time.time() - start_time
        
        # Bulk should be faster (though exact timing may vary)
        self.assertLess(bulk_time, individual_time * 2)  # Allow some margin
        
    def test_pagination_performance(self):
        """Test pagination performance with large datasets"""
        
        # Test different page sizes
        page_sizes = [10, 20, 50]
        
        for page_size in page_sizes:
            with self.assertQueryCount(1):
                books = frappe.get_all("Library Book",
                    fields=["name", "book_name"],
                    limit_start=0,
                    limit_page_length=page_size
                )
                self.assertEqual(len(books), min(page_size, 100))
                
    def test_caching_effectiveness(self):
        """Test that caching reduces database queries"""
        
        book_name = frappe.get_all("Library Book", limit=1, pluck="name")[0]
        
        # First access - should hit database
        with self.assertQueryCount(1):
            book1 = frappe.get_doc("Library Book", book_name)
            
        # Second access - should use cache (0 queries)
        with self.assertQueryCount(0):
            book2 = frappe.get_doc("Library Book", book_name)
            
        # Should be same object from cache
        self.assertEqual(book1.name, book2.name)
```

## Test Data Management

### Fixtures and Test Records

```python
# In your_app/fixtures/library_test_data.json
[
    {
        "doctype": "Author",
        "name": "test-author-1",
        "email": "author1@example.com", 
        "first_name": "John",
        "last_name": "Doe"
    },
    {
        "doctype": "Library Book",
        "name": "test-book-1",
        "book_name": "Python Programming",
        "author": "test-author-1",
        "isbn": "978-0-123456-789",
        "status": "Available"
    }
]

# In hooks.py
fixtures = [
    {"doctype": "Author", "filters": {"email": ["like", "%@example.com"]}},
    {"doctype": "Library Book", "filters": {"name": ["like", "test-%"]}}
]
```

### Dynamic Test Data Creation

```python
class TestDataFactory:
    """Factory class for creating test data"""
    
    @staticmethod
    def create_author(**kwargs):
        """Create test author with default values"""
        default_data = {
            "doctype": "Author",
            "email": frappe.mock("email"),
            "first_name": frappe.mock("first_name"),
            "last_name": frappe.mock("last_name")
        }
        default_data.update(kwargs)
        
        author = frappe.get_doc(default_data)
        author.insert(ignore_permissions=True)
        return author
        
    @staticmethod
    def create_book(**kwargs):
        """Create test book with default values"""
        # Ensure author exists
        if "author" not in kwargs:
            author = TestDataFactory.create_author()
            kwargs["author"] = author.name
            
        default_data = {
            "doctype": "Library Book",
            "book_name": frappe.mock("sentence"),
            "isbn": f"978-0-{frappe.generate_hash()[:10]}",
            "publication_year": frappe.utils.random.randint(2000, 2023),
            "status": "Available"
        }
        default_data.update(kwargs)
        
        book = frappe.get_doc(default_data)
        book.insert(ignore_permissions=True)
        return book
        
    @staticmethod
    def create_library_scenario():
        """Create a complete library scenario with multiple related records"""
        
        # Create authors
        authors = []
        for i in range(3):
            author = TestDataFactory.create_author()
            authors.append(author.name)
            
        # Create books
        books = []
        for i in range(10):
            book = TestDataFactory.create_book(
                author=authors[i % len(authors)],
                status=["Available", "Issued", "Reserved"][i % 3]
            )
            books.append(book.name)
            
        # Create members  
        members = []
        for i in range(5):
            member = frappe.get_doc({
                "doctype": "Library Member",
                "first_name": frappe.mock("first_name"),
                "last_name": frappe.mock("last_name"),
                "email": frappe.mock("email"),
                "membership_type": ["Basic", "Premium"][i % 2]
            })
            member.insert(ignore_permissions=True)
            members.append(member.name)
            
        return {
            "authors": authors,
            "books": books, 
            "members": members
        }
```

### Test Data Cleanup

```python
class TestLibraryWithCleanup(FrappeTestCase):
    def setUp(self):
        """Create test data before each test"""
        self.test_data = TestDataFactory.create_library_scenario()
        
    def tearDown(self):
        """Clean up test data after each test"""
        # Clean up in reverse dependency order
        
        # Delete library issues first
        frappe.db.delete("Library Issue", {
            "book": ["in", self.test_data["books"]]
        })
        
        # Delete books
        frappe.db.delete("Library Book", {
            "name": ["in", self.test_data["books"]]
        })
        
        # Delete members
        frappe.db.delete("Library Member", {
            "name": ["in", self.test_data["members"]]
        })
        
        # Delete authors
        frappe.db.delete("Author", {
            "name": ["in", self.test_data["authors"]]
        })
        
        frappe.db.commit()

# Context manager for temporary test data
@contextmanager
def temp_test_data():
    """Context manager for temporary test data creation and cleanup"""
    
    test_records = []
    
    try:
        # Create test data
        scenario = TestDataFactory.create_library_scenario()
        test_records.extend(scenario["books"])
        test_records.extend(scenario["members"])
        test_records.extend(scenario["authors"])
        
        yield scenario
        
    finally:
        # Cleanup
        for record_type in ["Library Book", "Library Member", "Author"]:
            frappe.db.delete(record_type, {"name": ["like", "test-%"]})
        frappe.db.commit()

# Usage
def test_with_temp_data(self):
    with temp_test_data() as data:
        # Use test data
        book = frappe.get_doc("Library Book", data["books"][0])
        self.assertEqual(book.status, "Available")
    # Data automatically cleaned up
```

## Mocking and Isolation

### External Service Mocking

```python
class TestExternalIntegrations(FrappeTestCase):
    @patch('frappe.utils.email.send_email')
    def test_book_reservation_notification(self, mock_send_email):
        """Test email notification with mocked email service"""
        
        # Create test data
        member = TestDataFactory.create_member()
        book = TestDataFactory.create_book()
        
        # Create reservation
        reservation = frappe.get_doc({
            "doctype": "Library Reservation",
            "member": member.name,
            "book": book.name,
            "reservation_date": frappe.utils.today()
        })
        reservation.insert()
        
        # Verify email was sent
        mock_send_email.assert_called_once()
        
        # Verify email content
        call_args = mock_send_email.call_args
        self.assertIn(member.email, call_args.kwargs.get("recipients", []))
        self.assertIn(book.book_name, call_args.kwargs.get("message", ""))
        
    @patch('requests.post')
    def test_external_api_webhook(self, mock_post):
        """Test webhook to external system"""
        
        mock_post.return_value.status_code = 200
        mock_post.return_value.json.return_value = {"success": True}
        
        # Create book that triggers webhook
        book = TestDataFactory.create_book()
        
        # Trigger webhook manually (normally automatic)
        frappe.enqueue("your_app.webhooks.send_book_data", book_name=book.name)
        
        # Process background jobs
        frappe.enqueue_background_jobs()
        
        # Verify external API was called
        mock_post.assert_called_once()
        
        # Verify payload structure
        call_data = mock_post.call_args.json
        self.assertEqual(call_data["book_name"], book.book_name)
        self.assertEqual(call_data["isbn"], book.isbn)
        
    @patch('frappe.utils.file_manager.save_file')
    def test_file_upload_handling(self, mock_save_file):
        """Test file upload with mocked file operations"""
        
        # Mock file save response
        mock_save_file.return_value = {
            "file_name": "test_book_cover.jpg",
            "file_url": "/files/test_book_cover.jpg",
            "file_size": 102400
        }
        
        # Test file attachment to book
        book = TestDataFactory.create_book()
        
        # Simulate file upload
        file_doc = frappe.get_doc({
            "doctype": "File",
            "file_name": "book_cover.jpg",
            "attached_to_doctype": "Library Book",
            "attached_to_name": book.name,
            "content": b"fake_image_data"
        })
        file_doc.insert()
        
        # Verify file was "saved"
        mock_save_file.assert_called_once()
```

### System Configuration Mocking

```python
from frappe.tests.utils import change_settings

class TestSystemConfiguration(FrappeTestCase):
    @change_settings("Library Settings", max_books_per_member=3)
    def test_book_limit_enforcement(self):
        """Test book borrowing limits with modified settings"""
        
        member = TestDataFactory.create_member()
        books = [TestDataFactory.create_book() for _ in range(5)]
        
        # Issue first 3 books (should succeed)
        for i in range(3):
            issue = frappe.get_doc({
                "doctype": "Library Issue",
                "member": member.name,
                "book": books[i].name,
                "issue_date": frappe.utils.today()
            })
            issue.insert()
            
        # Try to issue 4th book (should fail)
        with self.assertRaises(frappe.exceptions.ValidationError):
            issue = frappe.get_doc({
                "doctype": "Library Issue", 
                "member": member.name,
                "book": books[3].name,
                "issue_date": frappe.utils.today()
            })
            issue.insert()
            
    def test_temporary_system_settings(self):
        """Test with temporarily modified system settings"""
        
        original_value = frappe.get_single_value("Library Settings", "late_fee_per_day")
        
        try:
            # Temporarily change setting
            frappe.db.set_single_value("Library Settings", "late_fee_per_day", 5.0)
            
            # Test with new setting
            fee = frappe.get_single_value("Library Settings", "late_fee_per_day")
            self.assertEqual(fee, 5.0)
            
            # Test late fee calculation
            issue = frappe.get_doc({
                "doctype": "Library Issue",
                "member": TestDataFactory.create_member().name,
                "book": TestDataFactory.create_book().name,
                "issue_date": frappe.utils.add_days(frappe.utils.today(), -10),
                "return_date": frappe.utils.today()
            })
            
            late_fee = issue.calculate_late_fee()
            expected_fee = 10 * 5.0  # 10 days * 5.0 per day
            self.assertEqual(late_fee, expected_fee)
            
        finally:
            # Restore original setting
            frappe.db.set_single_value("Library Settings", "late_fee_per_day", original_value)
```

## Integration Testing

### End-to-End Workflows

```python
class TestLibraryIntegration(FrappeTestCase):
    def test_complete_library_workflow(self):
        """Test complete library workflow from member registration to book return"""
        
        # Step 1: Member registration
        member_data = {
            "doctype": "Library Member",
            "first_name": "John",
            "last_name": "Doe", 
            "email": "john.doe@example.com",
            "membership_type": "Premium"
        }
        
        member = frappe.get_doc(member_data)
        member.insert()
        
        # Verify member creation triggered user account creation
        user = frappe.get_doc("User", member.email)
        self.assertTrue(user)
        self.assertIn("Library Member", [role.role for role in user.roles])
        
        # Step 2: Book catalog search and reservation
        available_books = frappe.get_all("Library Book", 
            filters={"status": "Available"},
            limit=3
        )
        
        selected_book = available_books[0]
        
        reservation = frappe.get_doc({
            "doctype": "Library Reservation",
            "member": member.name,
            "book": selected_book.name,
            "reservation_date": frappe.utils.today()
        })
        reservation.insert()
        
        # Verify book status changed
        book = frappe.get_doc("Library Book", selected_book.name)
        self.assertEqual(book.status, "Reserved")
        
        # Step 3: Book issue
        issue = frappe.get_doc({
            "doctype": "Library Issue",
            "member": member.name,
            "book": selected_book.name,
            "issue_date": frappe.utils.today(),
            "due_date": frappe.utils.add_days(frappe.utils.today(), 14)
        })
        issue.insert()
        
        # Verify reservation was cancelled and book status changed
        book.reload()
        self.assertEqual(book.status, "Issued")
        
        reservation.reload()
        self.assertEqual(reservation.status, "Completed")
        
        # Step 4: Book return
        return_date = frappe.utils.add_days(issue.due_date, -2)  # Returned early
        issue.return_date = return_date
        issue.save()
        
        # Verify book became available again
        book.reload()
        self.assertEqual(book.status, "Available")
        
        # Step 5: Verify member activity log
        activities = frappe.get_all("Library Activity",
            filters={"member": member.name},
            fields=["activity_type", "book", "date"]
        )
        
        activity_types = [a.activity_type for a in activities]
        expected_activities = ["Reserved", "Issued", "Returned"]
        for activity in expected_activities:
            self.assertIn(activity, activity_types)
            
    def test_cross_module_integration(self):
        """Test integration between library and other modules"""
        
        # Create library member linked to employee
        employee = frappe.get_doc({
            "doctype": "Employee",
            "employee_name": "Jane Smith",
            "user_id": "jane.smith@company.com",
            "company": "Test Company"
        })
        employee.insert(ignore_permissions=True)
        
        member = frappe.get_doc({
            "doctype": "Library Member",
            "first_name": "Jane",
            "last_name": "Smith",
            "email": "jane.smith@company.com",
            "employee": employee.name,
            "membership_type": "Corporate"
        })
        member.insert()
        
        # Test that employee changes reflect in member
        employee.employee_name = "Jane Wilson"
        employee.save()
        
        member.reload()
        self.assertEqual(member.last_name, "Wilson")  # Should auto-update
        
        # Test department-based book recommendations
        employee.department = "IT"
        employee.save()
        
        recommendations = frappe.get_all("Library Book",
            filters={"category": "Technology", "status": "Available"},
            limit=5
        )
        
        # Should have technology book recommendations for IT department
        self.assertGreater(len(recommendations), 0)
```

### Multi-User Testing

```python
class TestMultiUserScenarios(FrappeTestCase):
    def test_concurrent_book_issue(self):
        """Test concurrent book issue requests"""
        
        # Create test data
        book = TestDataFactory.create_book()
        member1 = TestDataFactory.create_member()
        member2 = TestDataFactory.create_member()
        
        # Simulate concurrent requests using multiple connections
        with self.primary_connection():
            # First user starts transaction
            frappe.set_user(member1.email)
            issue1 = frappe.get_doc({
                "doctype": "Library Issue",
                "member": member1.name,
                "book": book.name,
                "issue_date": frappe.utils.today()
            })
            
            # Don't commit yet - simulate slow operation
            
        with self.secondary_connection():
            # Second user tries same book
            frappe.set_user(member2.email)
            issue2 = frappe.get_doc({
                "doctype": "Library Issue",
                "member": member2.name,
                "book": book.name,
                "issue_date": frappe.utils.today()
            })
            
            # This should fail - book already being issued
            with self.assertRaises(frappe.exceptions.ValidationError):
                issue2.insert()
                
        # Complete first transaction
        with self.primary_connection():
            issue1.insert()
            frappe.db.commit()
            
        # Verify first user got the book
        book.reload()
        self.assertEqual(book.status, "Issued")
        
    def test_permission_isolation(self):
        """Test that users can only access their own data"""
        
        # Create two members with different permissions
        member1 = TestDataFactory.create_member()
        member2 = TestDataFactory.create_member()
        
        # Member1 issues a book
        frappe.set_user(member1.email)
        book1 = TestDataFactory.create_book()
        issue1 = frappe.get_doc({
            "doctype": "Library Issue",
            "member": member1.name,
            "book": book1.name,
            "issue_date": frappe.utils.today()
        })
        issue1.insert()
        
        # Member2 should not see member1's issues
        frappe.set_user(member2.email)
        
        accessible_issues = frappe.get_all("Library Issue",
            filters={"member": member1.name}
        )
        self.assertEqual(len(accessible_issues), 0)  # Should be empty
        
        # Member2 should see their own issues
        book2 = TestDataFactory.create_book()
        issue2 = frappe.get_doc({
            "doctype": "Library Issue",
            "member": member2.name,
            "book": book2.name,
            "issue_date": frappe.utils.today()
        })
        issue2.insert()
        
        own_issues = frappe.get_all("Library Issue",
            filters={"member": member2.name}
        )
        self.assertEqual(len(own_issues), 1)
        
        # Reset to admin
        frappe.set_user("Administrator")
```

## Automated Testing Workflows

### Continuous Integration Setup

```yaml
# .github/workflows/test.yml
name: Library App Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mariadb:
        image: mariadb:10.6
        env:
          MARIADB_ROOT_PASSWORD: root
        options: --health-cmd="healthcheck.sh --connect --innodb_initialized" --health-interval=5s --health-timeout=2s --health-retries=3

    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.8
        
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: 14
        
    - name: Install Frappe
      run: |
        wget https://raw.githubusercontent.com/frappe/bench/develop/install.py
        python3 install.py --develop
        
    - name: Create test site
      run: |
        cd ~/frappe-bench
        bench new-site test_site --admin-password admin --mariadb-root-password root
        bench --site test_site install-app library_management
        
    - name: Run tests
      run: |
        cd ~/frappe-bench
        bench --site test_site run-tests --app library_management --coverage
        
    - name: Upload coverage
      uses: codecov/codecov-action@v1
      with:
        file: ~/frappe-bench/sites/coverage.xml
        flags: library_management
```

### Pre-commit Hooks

```python
# hooks.py
def run_tests_before_commit():
    """Run critical tests before allowing commit"""
    
    import subprocess
    import sys
    
    # Run unit tests
    result = subprocess.run([
        "bench", "--site", "test_site", "run-tests", 
        "--app", "library_management",
        "--module", "tests.test_critical"
    ], capture_output=True, text=True)
    
    if result.returncode != 0:
        print("Critical tests failed!")
        print(result.stdout)
        print(result.stderr)
        sys.exit(1)
        
    print("All critical tests passed!")
    
# In your git hooks or CI configuration
# call run_tests_before_commit()
```

### Test Reporting

```python
class TestReporter:
    """Generate test reports and metrics"""
    
    @staticmethod
    def generate_coverage_report():
        """Generate test coverage report"""
        
        import coverage
        
        cov = coverage.Coverage()
        cov.start()
        
        # Run tests
        frappe.test_runner.main()
        
        cov.stop()
        cov.save()
        
        # Generate reports
        cov.report()
        cov.html_report(directory='coverage_html')
        
    @staticmethod
    def performance_benchmark():
        """Run performance benchmarks"""
        
        import time
        import statistics
        
        benchmarks = {
            "book_creation": [],
            "member_search": [],
            "issue_processing": []
        }
        
        # Run each benchmark multiple times
        for _ in range(10):
            # Book creation benchmark
            start = time.time()
            TestDataFactory.create_book()
            benchmarks["book_creation"].append(time.time() - start)
            
            # Member search benchmark  
            start = time.time()
            frappe.get_all("Library Member", limit=50)
            benchmarks["member_search"].append(time.time() - start)
            
            # Issue processing benchmark
            start = time.time()
            member = TestDataFactory.create_member()
            book = TestDataFactory.create_book()
            issue = frappe.get_doc({
                "doctype": "Library Issue",
                "member": member.name,
                "book": book.name,
                "issue_date": frappe.utils.today()
            })
            issue.insert()
            benchmarks["issue_processing"].append(time.time() - start)
            
        # Calculate statistics
        results = {}
        for operation, times in benchmarks.items():
            results[operation] = {
                "mean": statistics.mean(times),
                "median": statistics.median(times),
                "stdev": statistics.stdev(times) if len(times) > 1 else 0
            }
            
        return results
```

## Debugging Techniques

### Test Debugging Tools

```python
class TestDebugger:
    """Utilities for debugging test failures"""
    
    @staticmethod
    def print_document_state(doc):
        """Print complete document state for debugging"""
        
        print(f"\n=== Document State: {doc.doctype} - {doc.name} ===")
        print(f"Docstatus: {doc.docstatus}")
        print(f"Creation: {doc.creation}")
        print(f"Modified: {doc.modified}")
        print(f"Owner: {doc.owner}")
        
        print("\n--- Field Values ---")
        for field in doc.meta.fields:
            if field.fieldtype not in ["Section Break", "Column Break"]:
                value = doc.get(field.fieldname)
                print(f"{field.fieldname}: {value} ({type(value).__name__})")
                
        print("\n--- Child Tables ---")
        for table_field in doc.meta.get_table_fields():
            child_docs = doc.get(table_field.fieldname)
            print(f"{table_field.fieldname}: {len(child_docs)} rows")
            for i, child in enumerate(child_docs):
                print(f"  Row {i}: {child.as_dict()}")
                
    @contextmanager
    def debug_queries(self):
        """Context manager to debug database queries"""
        
        queries = []
        
        def query_logger(query, values=None):
            queries.append({
                "query": query,
                "values": values,
                "timestamp": frappe.utils.now()
            })
            
        # Patch query execution
        original_sql = frappe.db.sql
        frappe.db.sql = lambda *args, **kwargs: (
            query_logger(args[0], args[1] if len(args) > 1 else None),
            original_sql(*args, **kwargs)
        )[1]
        
        try:
            yield queries
        finally:
            frappe.db.sql = original_sql
            
        print(f"\n=== Executed {len(queries)} queries ===")
        for i, q in enumerate(queries):
            print(f"{i+1}. {q['query'][:100]}...")
            if q['values']:
                print(f"   Values: {q['values']}")

    @staticmethod
    def debug_permissions(doctype, doc_name=None, user=None):
        """Debug permission checking process"""
        
        user = user or frappe.session.user
        
        print(f"\n=== Permission Debug: {doctype} for {user} ===")
        
        # User roles
        roles = frappe.get_roles(user)
        print(f"User roles: {roles}")
        
        # DocType permissions
        docperms = frappe.get_all("DocPerm", 
            filters={"parent": doctype, "role": ["in", roles]},
            fields=["role", "read", "write", "create", "delete", "permlevel"]
        )
        print(f"Role permissions: {docperms}")
        
        # User permissions
        user_perms = frappe.get_all("User Permission",
            filters={"user": user},
            fields=["allow", "for_value", "applicable_for"]
        )
        print(f"User restrictions: {user_perms}")
        
        # Check specific document
        if doc_name:
            doc = frappe.get_doc(doctype, doc_name)
            
            # Check each permission type
            for ptype in ["read", "write", "create", "delete", "submit", "cancel"]:
                has_perm = frappe.has_permission(doctype, ptype, doc, user)
                print(f"{ptype}: {'✓' if has_perm else '✗'}")

# Usage in tests
def test_with_debugging(self):
    """Example test with debugging utilities"""
    
    book = TestDataFactory.create_book()
    
    # Debug document state
    TestDebugger.print_document_state(book)
    
    # Debug queries during operation
    with TestDebugger().debug_queries() as queries:
        frappe.get_all("Library Book", filters={"status": "Available"})
        
    # Debug permissions
    TestDebugger.debug_permissions("Library Book", book.name)
```

## Best Practices

### Test Organization

```python
# Organize tests by functionality, not by file structure

class TestBookCRUD(FrappeTestCase):
    """Test basic CRUD operations for books"""
    pass
    
class TestBookValidation(FrappeTestCase):
    """Test book validation rules"""
    pass
    
class TestBookWorkflow(FrappeTestCase): 
    """Test book workflow state changes"""
    pass

class TestBookPermissions(FrappeTestCase):
    """Test book-related permissions"""
    pass

class TestBookIntegration(FrappeTestCase):
    """Test book integration with other modules"""
    pass
```

### Test Naming Conventions

```python
class TestLibraryBook(FrappeTestCase):
    # ✅ Good test names - descriptive and specific
    def test_create_book_with_valid_isbn_succeeds(self):
        pass
        
    def test_create_book_without_required_author_fails(self):
        pass
        
    def test_issue_available_book_changes_status_to_issued(self):
        pass
        
    def test_return_overdue_book_calculates_correct_late_fee(self):
        pass
        
    # ❌ Bad test names - vague and non-descriptive  
    def test_book(self):
        pass
        
    def test_validation(self):
        pass
        
    def test_status(self):
        pass
```

### Test Data Management

```python
class TestDataBestPractices(FrappeTestCase):
    
    def test_isolated_test_data(self):
        """Each test should create its own data to avoid interference"""
        
        # ✅ Good - create specific test data
        book = frappe.get_doc({
            "doctype": "Library Book",
            "book_name": "Test Specific Book",
            "author": "test@example.com",
            "isbn": "978-0-test-book"
        })
        book.insert()
        
        # Test logic here
        
        # Cleanup
        frappe.delete_doc("Library Book", book.name)
        
    def test_avoid_hard_coded_data(self):
        """Avoid depending on hard-coded existing data"""
        
        # ❌ Bad - depends on existing data that might not exist
        # existing_book = frappe.get_doc("Library Book", "BOOK-001")
        
        # ✅ Good - create or ensure data exists
        if not frappe.db.exists("Author", "test@example.com"):
            frappe.get_doc({
                "doctype": "Author",
                "email": "test@example.com",
                "first_name": "Test"
            }).insert()
            
        book = self.create_test_book(author="test@example.com")
        
    def test_minimal_test_data(self):
        """Create only the minimum data needed for the test"""
        
        # ✅ Good - minimal data for specific test
        book = frappe.get_doc({
            "doctype": "Library Book", 
            "book_name": "Test Book",
            "isbn": "978-0-minimal-test"
            # Only include fields needed for this test
        })
        book.insert(ignore_mandatory=True)  # Skip non-essential validations
```

### Assertion Best Practices

```python
class TestAssertionBestPractices(FrappeTestCase):
    
    def test_specific_assertions(self):
        """Use specific assertions with clear error messages"""
        
        book = self.create_test_book()
        
        # ✅ Good - specific assertions
        self.assertEqual(book.status, "Available", "New book should be available")
        self.assertIsNotNone(book.creation, "Book should have creation timestamp")
        self.assertIn("isbn", book.as_dict(), "Book should have ISBN field")
        
        # ❌ Bad - generic assertions
        # self.assertTrue(book.status == "Available")
        # self.assertTrue(book.creation)
        
    def test_exception_handling(self):
        """Test exceptions with specific exception types"""
        
        # ✅ Good - specific exception type and message check
        with self.assertRaises(frappe.exceptions.MandatoryError) as cm:
            frappe.get_doc({
                "doctype": "Library Book"
                # Missing required fields
            }).insert()
            
        self.assertIn("book_name", str(cm.exception))
        
        # ❌ Bad - generic exception handling
        # with self.assertRaises(Exception):
        #     frappe.get_doc({"doctype": "Library Book"}).insert()

    def test_multiple_assertions(self):
        """Group related assertions but keep them specific"""
        
        book = self.create_test_book()
        
        # Test document structure
        self.assertEqual(book.doctype, "Library Book")
        self.assertIsInstance(book.name, str)
        self.assertGreater(len(book.name), 0)
        
        # Test field values
        self.assertIsNotNone(book.book_name)
        self.assertIsNotNone(book.isbn)
        self.assertEqual(book.status, "Available")
        
        # Test metadata
        self.assertIsNotNone(book.creation)
        self.assertIsNotNone(book.modified)
        self.assertEqual(book.docstatus, 0)
```

---

**Next Steps**: Continue with [Database Operations](08-database-operations.md) to learn about ORM patterns, query optimization, schema management, and advanced database operations in Frappe applications.