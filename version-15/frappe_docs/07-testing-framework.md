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

### Transaction-Safe Document Creation Framework

For complex document creation scenarios, use transaction-safe patterns that provide immediate document visibility without commits while maintaining rollback capability:

```python
import frappe
import uuid
from typing import Dict, List, Any, Optional
from contextlib import contextmanager

class TransactionSafeDocumentFactory(FrappeTestCase):
    """
    Advanced document factory that creates complex hierarchies without commits
    while maintaining full rollback capability and immediate document visibility.
    
    Uses ERPNext-proven patterns for transaction-safe document creation.
    """
    
    def setUp(self):
        """Enhanced setUp with transaction-safe document management."""
        super().setUp()
        
        # Transaction state tracking
        self.savepoints = []
        self.created_documents = []
        self.document_cache = {}
        
        # Create initial savepoint for test isolation
        self.create_savepoint("test_start")
    
    def tearDown(self):
        """Transaction-safe tearDown with comprehensive cleanup."""
        try:
            # Restore user context first
            frappe.set_user("Administrator")
            
            # Use rollback for complete cleanup (ERPNext pattern)
            frappe.db.rollback()
            
        finally:
            # Clear caches
            self.document_cache.clear()
            self.created_documents.clear()
            self.savepoints.clear()
            
            # Call parent tearDown
            super().tearDown()
    
    def create_savepoint(self, name: str):
        """Create named savepoint for granular transaction control."""
        savepoint_name = f"{name}_{uuid.uuid4().hex[:8]}"
        frappe.db.savepoint(savepoint_name)
        self.savepoints.append(savepoint_name)
        return savepoint_name
    
    def rollback_to_savepoint(self, savepoint_name: str):
        """Rollback to specific savepoint."""
        frappe.db.rollback(save_point=savepoint_name)
        
        # Remove later savepoints
        try:
            idx = self.savepoints.index(savepoint_name)
            self.savepoints = self.savepoints[:idx + 1]
        except ValueError:
            pass
    
    def create_document_safe(self, doctype: str, doc_data: Dict[str, Any], 
                           cache_key: str = None) -> Any:
        """
        Create document with immediate visibility and rollback safety.
        
        Uses ERPNext production pattern: ignore_permissions=True for immediate visibility
        without requiring commits, while maintaining rollback capability.
        
        Args:
            doctype: DocType to create
            doc_data: Document data
            cache_key: Optional cache key for reuse
            
        Returns:
            Created document (immediately findable)
        """
        # Check cache first for reuse
        if cache_key and cache_key in self.document_cache:
            return self.document_cache[cache_key]
        
        # Check if document already exists (idempotent pattern)
        existing_doc = None
        if 'name' in doc_data:
            if frappe.db.exists(doctype, doc_data['name']):
                existing_doc = frappe.get_doc(doctype, doc_data['name'])
        
        if existing_doc:
            if cache_key:
                self.document_cache[cache_key] = existing_doc
            return existing_doc
        
        try:
            # Create document with immediate visibility (ERPNext pattern)
            doc_data.update({"doctype": doctype})
            document = frappe.get_doc(doc_data)
            
            # KEY: ignore_permissions=True makes document immediately findable
            document.insert(ignore_permissions=True)
            
            # Track for cleanup and cache
            self.created_documents.append((doctype, document.name))
            if cache_key:
                self.document_cache[cache_key] = document
            
            # Verify immediate visibility
            self.assertTrue(
                frappe.db.exists(doctype, document.name),
                f"Document {doctype} {document.name} should be immediately findable"
            )
            
            return document
            
        except Exception as e:
            frappe.log_error(
                title=f"Transaction-Safe Document Creation Failed: {doctype}",
                message=f"Error creating {doctype}: {str(e)}"
            )
            raise
    
    def create_user_safe(self, email: str, roles: List[str], 
                        first_name: str = "Test User") -> Any:
        """
        Create user with roles using transaction-safe pattern.
        
        Users are complex documents that need special handling for immediate visibility.
        """
        cache_key = f"user_{email}"
        
        if cache_key in self.document_cache:
            return self.document_cache[cache_key]
        
        # Check if user exists
        if frappe.db.exists("User", email):
            user = frappe.get_doc("User", email)
            self.document_cache[cache_key] = user
            return user
        
        try:
            # Create user document
            user_data = {
                "email": email,
                "first_name": first_name,
                "enabled": 1,
                "user_type": "System User",
                "new_password": "test123"  # Simple password for testing
            }
            
            user = self.create_document_safe("User", user_data, cache_key)
            
            # Add roles using transaction-safe pattern
            for role in roles:
                if role not in [r.role for r in user.roles]:
                    user.append('roles', {'role': role})
            
            # Save with ignore_permissions for immediate visibility
            user.save(ignore_permissions=True)
            
            # Verify user is immediately findable
            self.assertTrue(frappe.db.exists("User", email))
            
            return user
            
        except Exception as e:
            frappe.log_error(
                title=f"Transaction-Safe User Creation Failed: {email}",
                message=f"Error creating user {email}: {str(e)}"
            )
            raise
    
    def create_library_hierarchy_safe(self, base_name: str, 
                                    users: List[Dict[str, Any]] = None) -> Dict[str, Any]:
        """
        Create complete library hierarchy using transaction-safe patterns.
        
        Demonstrates complex multi-document creation without commits.
        """
        hierarchy = {}
        
        try:
            # 1. Create library with immediate visibility
            library_data = {
                "name": f"{base_name}_Library",
                "library_name": f"{base_name} Library",
                "description": f"Test library {base_name}"
            }
            
            library = self.create_document_safe(
                "Library", library_data, f"library_{base_name}"
            )
            hierarchy['library'] = library
            
            # 2. Create users and add as members
            if users:
                for user_config in users:
                    user = self.create_user_safe(
                        user_config['email'], 
                        user_config.get('roles', ['Library Member']),
                        user_config.get('name', 'Test User')
                    )
                    hierarchy.setdefault('users', []).append(user)
                    
                    # Add user to library members
                    library.append('members', {
                        'user': user.email,
                        'role': user_config.get('library_role', 'Member')
                    })
                
                # Save library with members (immediate visibility)
                library.save(ignore_permissions=True)
            
            # 3. Create books in library
            for i in range(3):
                book_data = {
                    "book_name": f"{base_name} Book {i+1}",
                    "library": library.name,
                    "author": f"author{i}@{base_name.lower()}.com",
                    "isbn": f"978-0-{base_name.lower()}-{i:03d}",
                    "status": "Available"
                }
                
                book = self.create_document_safe(
                    "Library Book", book_data, f"book_{base_name}_{i}"
                )
                hierarchy.setdefault('books', []).append(book)
            
            # Verify all documents are immediately findable (without commits!)
            self.assert_hierarchy_visibility(hierarchy)
            
            return hierarchy
            
        except Exception as e:
            frappe.log_error(
                title=f"Library Hierarchy Creation Failed: {base_name}",
                message=f"Error creating library hierarchy: {str(e)}"
            )
            raise
    
    def assert_hierarchy_visibility(self, hierarchy: Dict[str, Any]):
        """Assert that all created documents are immediately visible."""
        # Check library
        library = hierarchy['library']
        self.assertTrue(frappe.db.exists("Library", library.name))
        
        # Check books
        for book in hierarchy.get('books', []):
            self.assertTrue(frappe.db.exists("Library Book", book.name))
        
        # Check users
        for user in hierarchy.get('users', []):
            self.assertTrue(frappe.db.exists("User", user.email))
    
    def test_transaction_isolation_with_savepoints(self):
        """Demonstrate advanced transaction isolation using savepoints."""
        
        # Create initial state
        hierarchy1 = self.create_library_hierarchy_safe("TestLib1", [
            {'email': 'user1@testlib1.com', 'name': 'User 1', 'library_role': 'Librarian'}
        ])
        
        # Create savepoint before second library
        savepoint = self.create_savepoint("before_library_2")
        
        # Create second library
        hierarchy2 = self.create_library_hierarchy_safe("TestLib2", [
            {'email': 'user2@testlib2.com', 'name': 'User 2', 'library_role': 'Member'}
        ])
        
        # Verify both exist
        self.assertTrue(frappe.db.exists("Library", "TestLib1_Library"))
        self.assertTrue(frappe.db.exists("Library", "TestLib2_Library"))
        
        # Rollback to savepoint (removes library 2, keeps library 1)
        self.rollback_to_savepoint(savepoint)
        
        # Verify selective rollback
        self.assertTrue(frappe.db.exists("Library", "TestLib1_Library"))
        self.assertFalse(frappe.db.exists("Library", "TestLib2_Library"))
        
        return hierarchy1
    
    def test_complex_document_relationships_without_commits(self):
        """Test complex document relationships using transaction-safe patterns."""
        
        # Create complete library system (no commits needed!)
        hierarchy = self.create_library_hierarchy_safe("ComplexTest", [
            {'email': 'librarian@complex.test', 'name': 'Head Librarian', 'library_role': 'Librarian'},
            {'email': 'member1@complex.test', 'name': 'Member One', 'library_role': 'Member'},
            {'email': 'member2@complex.test', 'name': 'Member Two', 'library_role': 'Member'}
        ])
        
        # All documents immediately findable without commits
        library = hierarchy['library']
        users = hierarchy['users']
        books = hierarchy['books']
        
        # Test complex operations on created hierarchy
        self.assertEqual(len(library.members), 3)
        self.assertEqual(len(books), 3)
        
        # Test dependency relationships work correctly
        for book in books:
            self.assertEqual(book.library, library.name)
            self.assertTrue(frappe.db.exists("Library", book.library))
        
        # Test user permissions work immediately (no commit needed)
        librarian = next(u for u in users if u.email == 'librarian@complex.test')
        frappe.set_user(librarian.email)
        
        # User can access library immediately (no commit needed)
        retrieved_library = frappe.get_doc("Library", library.name)
        self.assertEqual(retrieved_library.name, library.name)
        
        # Restore admin context
        frappe.set_user("Administrator")
        
        # Everything will be cleaned up by rollback in tearDown!
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

## Background Job Testing Framework

For testing asynchronous operations and background jobs, Frappe provides comprehensive testing patterns that enable validation of complex background processes:

```python
import frappe
from unittest.mock import patch, MagicMock
from contextlib import contextmanager
import time
import threading
from queue import Queue, Empty
from typing import Callable, Dict, List, Any, Optional
import inspect

class BackgroundJobTestFramework:
    """
    Comprehensive framework for testing background job execution.
    
    Provides utilities for:
    - Immediate execution testing
    - Delayed execution simulation
    - Job queue monitoring
    - Async operation validation
    - Job failure testing
    """
    
    def __init__(self):
        self.job_queue = Queue()
        self.executed_jobs = []
        self.job_results = {}
        self.job_errors = {}
        
    @contextmanager
    def immediate_job_execution(self):
        """
        Context manager that executes background jobs immediately for testing.
        
        This is the most useful mode for integration testing where you want
        to validate that background operations complete successfully.
        """
        original_enqueue = frappe.utils.background_jobs.enqueue
        
        def immediate_execute(method, **kwargs):
            """Execute job immediately instead of queuing."""
            try:
                # Extract job function
                if isinstance(method, str):
                    module_path, function_name = method.rsplit('.', 1)
                    module = __import__(module_path, fromlist=[function_name])
                    job_function = getattr(module, function_name)
                else:
                    job_function = method
                
                # Extract method arguments
                job_args = {}
                job_kwargs = {}
                
                # Filter out enqueue-specific arguments
                enqueue_args = {'queue', 'timeout', 'job_name', 'is_async', 'now', 'enqueue_after', 'job_id'}
                for key, value in kwargs.items():
                    if key not in enqueue_args:
                        job_kwargs[key] = value
                
                # Execute immediately
                result = job_function(**job_kwargs)
                
                # Track execution
                job_id = f"immediate_{len(self.executed_jobs)}"
                self.executed_jobs.append({
                    'job_id': job_id,
                    'method': method,
                    'kwargs': job_kwargs,
                    'result': result,
                    'execution_time': time.time()
                })
                
                return job_id
                
            except Exception as e:
                job_id = f"failed_{len(self.job_errors)}"
                self.job_errors[job_id] = e
                raise
        
        # Patch the enqueue function
        with patch('frappe.utils.background_jobs.enqueue', side_effect=immediate_execute):
            try:
                yield self
            finally:
                # Restore original function
                frappe.utils.background_jobs.enqueue = original_enqueue
    
    @contextmanager
    def monitored_job_execution(self, execution_delay: float = 0.1):
        """
        Context manager that monitors background job execution.
        
        Args:
            execution_delay: Delay before executing jobs (simulates queue processing)
        """
        job_monitor = JobExecutionMonitor(self.job_queue, execution_delay)
        
        def monitored_enqueue(method, **kwargs):
            """Queue job for monitored execution."""
            job_id = f"monitored_{int(time.time() * 1000)}"
            
            self.job_queue.put({
                'job_id': job_id,
                'method': method,
                'kwargs': kwargs,
                'queued_at': time.time()
            })
            
            return job_id
        
        # Start monitoring thread
        monitor_thread = threading.Thread(target=job_monitor.run, daemon=True)
        monitor_thread.start()
        
        with patch('frappe.utils.background_jobs.enqueue', side_effect=monitored_enqueue):
            try:
                yield job_monitor
            finally:
                job_monitor.stop()
                monitor_thread.join(timeout=5)
    
    def get_job_execution_report(self) -> Dict[str, Any]:
        """Get comprehensive report of job executions."""
        return {
            'total_jobs_executed': len(self.executed_jobs),
            'successful_jobs': len([j for j in self.executed_jobs if 'error' not in j]),
            'failed_jobs': len(self.job_errors),
            'execution_details': self.executed_jobs,
            'error_details': self.job_errors
        }


class JobExecutionMonitor:
    """Monitor and execute queued background jobs."""
    
    def __init__(self, job_queue: Queue, execution_delay: float = 0.1):
        self.job_queue = job_queue
        self.execution_delay = execution_delay
        self.running = True
        self.results = {}
    
    def run(self):
        """Run the job monitoring loop."""
        while self.running:
            try:
                # Get job from queue
                job = self.job_queue.get(timeout=0.1)
                
                # Wait for execution delay
                if self.execution_delay > 0:
                    time.sleep(self.execution_delay)
                
                # Execute job
                self._execute_job(job)
                
            except Empty:
                continue
            except Exception as e:
                frappe.log_error(
                    title="Job Monitor Error",
                    message=f"Error in job monitor: {str(e)}"
                )
    
    def _execute_job(self, job: Dict[str, Any]):
        """Execute a single job."""
        try:
            method = job['method']
            kwargs = job.get('kwargs', {})
            job_id = job['job_id']
            
            # Extract job function
            if isinstance(method, str):
                module_path, function_name = method.rsplit('.', 1)
                module = __import__(module_path, fromlist=[function_name])
                job_function = getattr(module, function_name)
            else:
                job_function = method
            
            # Filter arguments
            enqueue_args = {'queue', 'timeout', 'job_name', 'is_async', 'now', 'enqueue_after', 'job_id'}
            filtered_kwargs = {k: v for k, v in kwargs.items() if k not in enqueue_args}
            
            # Execute
            result = job_function(**filtered_kwargs)
            
            # Store result
            self.results[job_id] = {
                'status': 'success',
                'result': result,
                'executed_at': time.time()
            }
            
        except Exception as e:
            self.results[job_id] = {
                'status': 'error',
                'error': str(e),
                'executed_at': time.time()
            }
    
    def stop(self):
        """Stop the job monitor."""
        self.running = False
    
    def wait_for_jobs(self, timeout: float = 10.0) -> bool:
        """
        Wait for all queued jobs to complete.
        
        Args:
            timeout: Maximum time to wait
            
        Returns:
            True if all jobs completed, False if timeout
        """
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            if self.job_queue.empty():
                return True
            time.sleep(0.1)
        
        return False


# Integration with existing test framework
def background_job_test(execution_mode: str = "immediate"):
    """
    Decorator for tests that involve background jobs.
    
    Args:
        execution_mode: "immediate", "monitored", or "disabled"
    """
    def decorator(test_func):
        def wrapper(self):
            framework = BackgroundJobTestFramework()
            
            if execution_mode == "immediate":
                with framework.immediate_job_execution() as job_context:
                    result = test_func(self, job_context)
                    
                    # Add job execution report to test
                    if hasattr(self, 'background_job_report'):
                        self.background_job_report = framework.get_job_execution_report()
                    
                    return result
                    
            elif execution_mode == "monitored":
                with framework.monitored_job_execution() as job_monitor:
                    result = test_func(self, job_monitor)
                    
                    # Wait for jobs to complete
                    job_monitor.wait_for_jobs(timeout=10.0)
                    
                    return result
            else:
                # Disabled mode - no background job execution
                return test_func(self)
        
        return wrapper
    return decorator


# Example Usage: Testing Background Jobs
class TestLibraryBackgroundJobs(FrappeTestCase):
    """Test library system background job operations."""
    
    @background_job_test(execution_mode="immediate")
    def test_book_reservation_notification_job(self, job_context):
        """Test that book reservation triggers notification background job."""
        
        # Create test data
        member = self.create_test_member()
        book = self.create_test_book()
        
        # Create reservation (triggers background notification job)
        reservation = frappe.get_doc({
            "doctype": "Library Reservation",
            "member": member.name,
            "book": book.name,
            "reservation_date": frappe.utils.today()
        })
        reservation.insert()
        
        # Verify background job was executed
        job_report = job_context.get_job_execution_report()
        self.assertGreater(job_report['total_jobs_executed'], 0)
        
        # Verify notification was sent (would be mocked in real test)
        executed_jobs = job_report['execution_details']
        notification_jobs = [j for j in executed_jobs if 'notification' in str(j['method']).lower()]
        self.assertGreater(len(notification_jobs), 0)
    
    @background_job_test(execution_mode="monitored")
    def test_bulk_book_processing_job(self, job_monitor):
        """Test bulk book processing with monitored execution."""
        
        # Create multiple books that will trigger background processing
        books = []
        for i in range(5):
            book = self.create_test_book(book_name=f"Bulk Test Book {i}")
            books.append(book)
        
        # Trigger bulk processing job
        frappe.enqueue(
            'library_app.background_jobs.process_bulk_books',
            book_names=[book.name for book in books],
            queue='long'
        )
        
        # Wait for jobs to complete
        jobs_completed = job_monitor.wait_for_jobs(timeout=15.0)
        self.assertTrue(jobs_completed, "Background jobs should complete within timeout")
        
        # Verify job results
        self.assertGreater(len(job_monitor.results), 0)
        for job_id, result in job_monitor.results.items():
            self.assertEqual(result['status'], 'success')
    
    def test_job_failure_handling(self):
        """Test handling of background job failures."""
        
        with patch('frappe.utils.background_jobs.enqueue') as mock_enqueue:
            # Mock job failure
            mock_enqueue.side_effect = Exception("Queue service unavailable")
            
            # Create operation that would trigger background job
            member = self.create_test_member()
            book = self.create_test_book()
            
            # This should handle job failure gracefully
            try:
                reservation = frappe.get_doc({
                    "doctype": "Library Reservation",
                    "member": member.name,
                    "book": book.name,
                    "reservation_date": frappe.utils.today()
                })
                reservation.insert()
                
                # Reservation should still be created even if job fails
                self.assertTrue(frappe.db.exists("Library Reservation", reservation.name))
                
            except Exception:
                self.fail("Document creation should not fail due to background job issues")
    
    @background_job_test(execution_mode="immediate")
    def test_job_with_complex_dependencies(self, job_context):
        """Test background job that has complex document dependencies."""
        
        # Create library hierarchy
        library = self.create_document_safe("Library", {
            "library_name": "Background Job Test Library"
        })
        
        users = []
        for i in range(3):
            user = self.create_user_safe(
                f"user{i}@bgjob.test", 
                ['Library Member']
            )
            users.append(user)
        
        # Trigger job that processes all library members
        frappe.enqueue(
            'library_app.background_jobs.notify_all_library_members',
            library_name=library.name,
            message="Library maintenance scheduled"
        )
        
        # Verify job executed and processed all users
        job_report = job_context.get_job_execution_report()
        self.assertGreater(job_report['total_jobs_executed'], 0)
        
        # In real implementation, would verify each user got notification
        executed_jobs = job_report['execution_details']
        member_notification_job = next(
            (j for j in executed_jobs if 'notify_all_library_members' in str(j['method'])), 
            None
        )
        self.assertIsNotNone(member_notification_job)
        self.assertEqual(member_notification_job['kwargs']['library_name'], library.name)
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

### Advanced Dynamic Integration Mock Framework

For complex multi-service integration scenarios, Frappe provides an advanced mock framework that supports dynamic mock discovery and stateful integration testing:

```python
import frappe
import importlib
import inspect
from unittest.mock import MagicMock, patch, Mock
from contextlib import contextmanager
from typing import Dict, List, Optional, Any, Union, Callable

class IntegrationMockFramework:
    """
    Advanced mocking framework for complex multi-DocType integration testing.
    
    Features:
    - Dynamic mock target discovery
    - Multi-service integration mock management
    - Hierarchical mock inheritance
    - Context-aware mock validation
    - Integration state management
    """
    
    def __init__(self):
        self._mock_registry = {}
        self._active_mocks = {}
        self._mock_states = {}
        self._integration_contexts = {}
        self._discovered_targets = {}
        
    def discover_mock_targets(self, service_patterns: List[str]) -> Dict[str, List[str]]:
        """
        Dynamically discover mockable targets based on patterns.
        
        Args:
            service_patterns: List of module patterns to search
            
        Returns:
            Dictionary of service names to available mock targets
        """
        discovered = {}
        
        for pattern in service_patterns:
            try:
                # Dynamic import and inspection
                module_parts = pattern.split('.')
                for i in range(len(module_parts)):
                    try:
                        module_path = '.'.join(module_parts[:i+1])
                        module = importlib.import_module(module_path)
                        
                        # Find callable functions
                        functions = [
                            name for name, obj in inspect.getmembers(module, inspect.isfunction)
                            if not name.startswith('_')
                        ]
                        
                        if functions:
                            discovered[module_path] = functions
                            
                    except ImportError:
                        continue
                        
            except Exception as e:
                frappe.log_error(
                    title=f"Mock Target Discovery Error: {pattern}",
                    message=f"Failed to discover targets for {pattern}: {str(e)}"
                )
        
        self._discovered_targets.update(discovered)
        return discovered
    
    def register_integration_mock_config(self, integration_name: str, 
                                       mock_config: Dict[str, Any]) -> None:
        """
        Register comprehensive integration mock configuration.
        
        Args:
            integration_name: Name of the integration (e.g., "library_notifications")
            mock_config: Complex mock configuration with multiple services
        """
        self._mock_registry[integration_name] = {
            'services': mock_config.get('services', {}),
            'dependencies': mock_config.get('dependencies', []),
            'state_management': mock_config.get('state_management', {}),
            'validation_rules': mock_config.get('validation_rules', {}),
            'integration_points': mock_config.get('integration_points', [])
        }
    
    @contextmanager
    def mock_integration(self, integration_name: str, scenario: str = "default",
                        state_overrides: Dict[str, Any] = None):
        """
        Context manager for complex integration mocking.
        
        Args:
            integration_name: Name of the registered integration
            scenario: Scenario to activate
            state_overrides: Custom state overrides for this context
        """
        if integration_name not in self._mock_registry:
            raise KeyError(f"Integration '{integration_name}' not registered")
        
        config = self._mock_registry[integration_name]
        applied_patches = []
        integration_state = {}
        
        try:
            # Set up integration state
            integration_state = self._setup_integration_state(
                config, scenario, state_overrides
            )
            
            # Apply service mocks
            for service_name, service_config in config['services'].items():
                patches = self._apply_service_mocks(
                    service_name, service_config, scenario, integration_state
                )
                applied_patches.extend(patches)
            
            # Store active integration context
            self._integration_contexts[integration_name] = {
                'scenario': scenario,
                'state': integration_state,
                'patches': applied_patches
            }
            
            yield integration_state
            
        finally:
            # Cleanup
            for patch_obj in applied_patches:
                try:
                    patch_obj.stop()
                except Exception:
                    pass
                    
            # Clear integration context
            self._integration_contexts.pop(integration_name, None)

# Example Usage: Library System Integration Mocking
class TestLibraryIntegrationMocks(FrappeTestCase):
    def setUp(self):
        """Set up integration mock framework"""
        super().setUp()
        self.mock_framework = IntegrationMockFramework()
        
        # Register library system integration mocks
        library_config = {
            'services': {
                'notification_service': {
                    'scenarios': {
                        'success': {
                            'frappe.sendmail': MagicMock(return_value=True),
                            'frappe.publish_realtime': MagicMock(return_value=True),
                        },
                        'email_failure': {
                            'frappe.sendmail': MagicMock(side_effect=Exception("Email service down")),
                            'frappe.publish_realtime': MagicMock(return_value=True),
                        }
                    }
                },
                'external_apis': {
                    'scenarios': {
                        'success': {
                            'library_app.integrations.book_catalog.search_books': 
                                MagicMock(return_value={'books': [{'isbn': '123', 'title': 'Test Book'}]}),
                        },
                        'api_timeout': {
                            'library_app.integrations.book_catalog.search_books': 
                                MagicMock(side_effect=Exception("API timeout")),
                        }
                    }
                }
            },
            'state_management': {
                'default': {
                    'notifications_sent': 0,
                    'api_calls_made': 0,
                    'errors_encountered': 0
                }
            }
        }
        
        self.mock_framework.register_integration_mock_config('library_system', library_config)
    
    def test_book_issue_with_successful_notifications(self):
        """Test book issue with successful notification integration"""
        
        with self.mock_framework.mock_integration('library_system', 'success') as state:
            # Create test data
            member = self.create_test_member()
            book = self.create_test_book()
            
            # Issue book (triggers notifications)
            issue = frappe.get_doc({
                "doctype": "Library Issue",
                "member": member.name,
                "book": book.name,
                "issue_date": frappe.utils.today()
            })
            issue.insert()
            
            # Verify integration state was updated
            self.assertEqual(state.get('notifications_sent', 0), 1)

# Global instance for reuse
integration_mock_framework = IntegrationMockFramework()
```

### Realistic User Permission Testing

Critical gap identified: Tests typically run as Administrator but production users have limited permissions. This framework enables testing with actual user roles:

```python
import frappe
from frappe.tests.utils import FrappeTestCase
from contextlib import contextmanager
from typing import Dict, List, Any, Optional
import uuid

class RealisticUserTestingFramework(FrappeTestCase):
    """
    Enhanced base class for realistic user permission testing.
    
    Provides comprehensive utilities for testing with actual user roles and permissions
    instead of Administrator privileges, matching real-world usage patterns.
    """
    
    def setUp(self):
        """Enhanced setUp with realistic user testing support."""
        super().setUp()
        
        # Store original user context
        self.original_user = frappe.session.user
        
        # Initialize user management
        self.test_users = {}
        self.user_permissions = {}
        
        # Create standard test users for common scenarios
        self._create_standard_test_users()
    
    def tearDown(self):
        """Enhanced tearDown with proper user context restoration."""
        try:
            # Restore original user context
            frappe.set_user(self.original_user)
            
            # Clean up test users and permissions
            self._cleanup_test_users()
            
        finally:
            # Call parent tearDown
            super().tearDown()
    
    def _create_standard_test_users(self):
        """Create standard test users for common testing scenarios."""
        
        # Standard user types for testing
        standard_users = {
            'library_manager': {
                'email': f'lib_manager_{uuid.uuid4().hex[:8]}@test.com',
                'roles': ['Library Manager', 'Library Member'],
                'description': 'Library manager with full library permissions'
            },
            'library_member': {
                'email': f'lib_member_{uuid.uuid4().hex[:8]}@test.com', 
                'roles': ['Library Member'],
                'description': 'Regular library member with standard permissions'
            },
            'guest_user': {
                'email': f'guest_{uuid.uuid4().hex[:8]}@test.com',
                'roles': ['Guest'],
                'description': 'Guest user with read-only access'
            },
            'external_user': {
                'email': f'external_{uuid.uuid4().hex[:8]}@test.com',
                'roles': ['Desk User'],  # No library roles
                'description': 'External user with no library access'
            }
        }
        
        for user_type, config in standard_users.items():
            try:
                # Create user with roles
                user = self.create_test_user_with_roles(
                    config['email'], 
                    config['roles'],
                    first_name=f"Test {user_type.title().replace('_', ' ')}"
                )
                
                self.test_users[user_type] = user
                
            except Exception as e:
                frappe.log_error(
                    title=f"Failed to create test user: {user_type}",
                    message=f"Error: {str(e)}"
                )
    
    def create_test_user_with_roles(self, email: str, roles: List[str], 
                                  first_name: str = "Test User") -> Any:
        """
        Create a test user with specified roles.
        
        Args:
            email: User email address
            roles: List of role names to assign
            first_name: User's first name
            
        Returns:
            Created user document
        """
        try:
            # Check if user already exists
            if frappe.db.exists("User", email):
                user = frappe.get_doc("User", email)
            else:
                # Create new user
                user = frappe.get_doc({
                    "doctype": "User",
                    "email": email,
                    "first_name": first_name,
                    "enabled": 1,
                    "new_password": "test123",  # Simple password for testing
                    "user_type": "System User"
                })
                user.insert(ignore_permissions=True)
            
            # Add roles
            existing_roles = [role.role for role in user.roles]
            for role in roles:
                if role not in existing_roles:
                    user.append('roles', {
                        'role': role
                    })
            
            user.save(ignore_permissions=True)
            return user
            
        except Exception as e:
            frappe.log_error(
                title=f"User Creation Failed: {email}",
                message=f"Error creating user with roles {roles}: {str(e)}"
            )
            raise
    
    @contextmanager
    def test_with_user_context(self, user_type: str):
        """
        Context manager for testing with specific user permissions.
        
        Args:
            user_type: Type of user to use for testing
        """
        if user_type not in self.test_users:
            raise ValueError(f"User type '{user_type}' not found in standard test users")
        
        user = self.test_users[user_type]
        original_user = frappe.session.user
        
        try:
            # Set user context
            frappe.set_user(user.email)
            yield user
        finally:
            # Restore original user context
            frappe.set_user(original_user)
    
    def assert_permission_denied(self, operation_func, *args, **kwargs):
        """
        Assert that an operation raises permission error.
        
        Args:
            operation_func: Function to test
            *args, **kwargs: Arguments for the function
        """
        with self.assertRaises((frappe.PermissionError, frappe.ValidationError)):
            operation_func(*args, **kwargs)
    
    def assert_permission_allowed(self, operation_func, *args, **kwargs):
        """
        Assert that an operation succeeds (no permission error).
        
        Args:
            operation_func: Function to test
            *args, **kwargs: Arguments for the function
            
        Returns:
            Result of the operation
        """
        try:
            return operation_func(*args, **kwargs)
        except (frappe.PermissionError, frappe.ValidationError) as e:
            self.fail(f"Operation {operation_func.__name__} should have been allowed but raised: {str(e)}")

# Example Usage: Testing with Realistic User Permissions
class TestLibraryPermissions(RealisticUserTestingFramework):
    """Test library operations with realistic user permissions."""
    
    def test_book_access_control(self):
        """Test that users can only access books they have permission for."""
        
        # Create test book as Administrator
        frappe.set_user("Administrator")
        book = self.create_test_book()
        
        # Test 1: Library member can read books (realistic scenario)
        with self.test_with_user_context('library_member'):
            retrieved_book = self.assert_permission_allowed(
                frappe.get_doc, "Library Book", book.name
            )
            self.assertEqual(retrieved_book.name, book.name)
        
        # Test 2: External user cannot access library books (security validation)
        with self.test_with_user_context('external_user'):
            self.assert_permission_denied(
                frappe.get_doc, "Library Book", book.name
            )
    
    def test_book_creation_permissions(self):
        """Test book creation with different user permission levels."""
        
        book_data = {
            "doctype": "Library Book",
            "book_name": "Permission Test Book",
            "author": "test-author@example.com",
            "isbn": "978-0-permission-test"
        }
        
        # Test 1: Library manager can create books
        with self.test_with_user_context('library_manager'):
            book = self.assert_permission_allowed(
                frappe.get_doc(book_data).insert
            )
            self.assertEqual(book.book_name, "Permission Test Book")
        
        # Test 2: Regular member cannot create books
        with self.test_with_user_context('library_member'):
            self.assert_permission_denied(
                frappe.get_doc(book_data).insert
            )
        
        # Test 3: External user cannot create books
        with self.test_with_user_context('external_user'):
            self.assert_permission_denied(
                frappe.get_doc(book_data).insert
            )
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