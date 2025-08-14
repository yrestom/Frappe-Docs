# ERPNext Testing Methodologies Guide

## Table of Contents
- [Overview](#overview)
- [Testing Philosophy](#testing-philosophy)
- [Test Structure & Organization](#test-structure--organization)
- [Unit Testing Patterns](#unit-testing-patterns)
- [Integration Testing](#integration-testing)
- [Performance Testing](#performance-testing)
- [API Testing](#api-testing)
- [Report Testing](#report-testing)
- [Security Testing](#security-testing)
- [UI/UX Testing](#uiux-testing)
- [Test Data Management](#test-data-management)
- [Continuous Testing](#continuous-testing)
- [Testing Best Practices](#testing-best-practices)
- [Debugging & Troubleshooting](#debugging--troubleshooting)

## Overview

This comprehensive guide documents ERPNext's proven testing methodologies extracted from production codebase. These patterns ensure reliability, maintainability, and quality across all custom applications built on the Frappe Framework.

### ERPNext Testing Ecosystem
- **Framework**: Built on Python `unittest` with Frappe extensions
- **Coverage**: Unit, integration, performance, and security testing
- **Automation**: CI/CD integrated testing pipelines
- **Data Management**: Savepoint/rollback transaction management
- **Isolation**: Test-specific databases and environments

## Testing Philosophy

### 1. Test-Driven Quality
```python
# ERPNext testing principles:
# 1. Every feature requires corresponding tests
# 2. Tests must be isolated and repeatable
# 3. Business logic validation through comprehensive scenarios
# 4. Performance regression prevention
# 5. Security vulnerability prevention
```

### 2. Layered Testing Strategy
- **Unit Tests**: Individual component validation
- **Integration Tests**: Cross-module workflow validation  
- **System Tests**: End-to-end business process validation
- **Performance Tests**: Scalability and efficiency validation
- **Security Tests**: Permission and vulnerability validation

## Test Structure & Organization

### 1. Standard Test File Structure
```python
# From erpnext/tests/test_point_of_sale.py:14
import unittest
import frappe
from frappe.tests.utils import FrappeTestCase

class TestPointOfSale(unittest.TestCase):
    """
    Comprehensive POS functionality testing
    
    Test Coverage:
    - Item search and filtering
    - Stock validation
    - Price list integration
    - Transaction processing
    """
    
    @classmethod
    def setUpClass(cls) -> None:
        """Set up class-level test data"""
        frappe.db.savepoint("before_test_point_of_sale")
        cls.setup_test_data()
    
    @classmethod
    def tearDownClass(cls) -> None:
        """Clean up class-level test data"""
        frappe.db.rollback(save_point="before_test_point_of_sale")
    
    def setUp(self):
        """Set up individual test case"""
        self.test_company = "_Test Company"
        self.test_warehouse = "_Test Warehouse - _TC"
    
    def tearDown(self):
        """Clean up individual test case"""
        # Individual test cleanup if needed
        pass
    
    def test_item_search(self):
        """Test item search functionality with stock validation"""
        # Test implementation follows
```

### 2. Test Organization Patterns
```python
# Directory structure pattern:
# erpnext/
#   ├── tests/                    # Global integration tests
#   │   ├── test_point_of_sale.py
#   │   ├── test_perf.py         
#   │   └── test_notifications.py
#   ├── selling/
#   │   ├── doctype/
#   │   │   └── customer/
#   │   │       └── test_customer.py  # Unit tests
#   │   └── report/
#   │       └── sales_analytics/
#   │           └── test_analytics.py # Report tests
#   └── support/
#       └── doctype/
#           └── service_level_agreement/
#               └── test_service_level_agreement.py
```

### 3. Test Naming Conventions
```python
class TestServiceLevelAgreement(unittest.TestCase):
    """
    Naming Convention:
    - Class: Test{DocTypeName} or Test{ModuleName}
    - Methods: test_{specific_functionality}
    - Descriptive docstrings for each test
    """
    
    def test_default_service_level_agreement(self):
        """Test default SLA assignment and validation"""
        pass
    
    def test_customer_specific_sla(self):
        """Test customer-specific SLA override behavior"""
        pass
    
    def test_sla_resolution_time_calculation(self):
        """Test SLA resolution time computation with holidays"""
        pass
```

## Unit Testing Patterns

### 1. Document Testing Pattern
```python
# From erpnext/support/doctype/service_level_agreement/test_service_level_agreement.py
class TestServiceLevelAgreement(unittest.TestCase):
    
    def setUp(self):
        """Set up test environment"""
        self.create_company()
        frappe.db.set_single_value("Support Settings", "track_service_level_agreement", 1)
        
        # Clean up previous test data
        lead = frappe.qb.DocType("Lead")
        frappe.qb.from_(lead).delete().where(lead.company == self.company).run()
    
    def create_company(self):
        """Create test company or use existing"""
        name = "_Test Support SLA"
        
        if frappe.db.exists("Company", name):
            company = frappe.get_doc("Company", name)
        else:
            company = frappe.get_doc({
                "doctype": "Company",
                "company_name": name,
                "country": "India",
                "default_currency": "INR",
                "create_chart_of_accounts_based_on": "Standard Template",
                "chart_of_accounts": "Standard"
            })
            company = company.save()
        
        self.company = company.name
    
    def test_service_level_agreement(self):
        """Test SLA creation and retrieval patterns"""
        
        # Create default SLA
        default_sla = create_service_level_agreement(
            default_service_level_agreement=1,
            holiday_list="__Test Holiday List",
            entity_type=None,
            entity=None,
            response_time=14400,  # 4 hours
            resolution_time=21600  # 6 hours
        )
        
        # Retrieve and validate
        retrieved_sla = get_service_level_agreement(default_service_level_agreement=1)
        
        self.assertEqual(default_sla.name, retrieved_sla.name)
        self.assertEqual(default_sla.entity_type, retrieved_sla.entity_type)
        self.assertEqual(default_sla.default_service_level_agreement, 
                        retrieved_sla.default_service_level_agreement)

def create_service_level_agreement(**kwargs):
    """Helper function to create test SLA"""
    sla = frappe.get_doc({
        "doctype": "Service Level Agreement",
        "service_level": "Test SLA",
        **kwargs
    })
    
    # Add service level priority
    sla.append("priorities", {
        "priority": "High",
        "response_time": kwargs.get("response_time", 3600),
        "resolution_time": kwargs.get("resolution_time", 7200)
    })
    
    return sla.insert()
```

### 2. Validation Testing Pattern
```python
class TestDocumentValidation(FrappeTestCase):
    """Test document validation logic"""
    
    def test_required_field_validation(self):
        """Test required field validation"""
        with self.assertRaises(frappe.ValidationError) as context:
            doc = frappe.get_doc({
                "doctype": "Customer",
                # Missing required customer_name field
            })
            doc.insert()
        
        self.assertIn("customer_name", str(context.exception))
    
    def test_duplicate_prevention(self):
        """Test duplicate record prevention"""
        # Create first customer
        customer1 = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": "Test Customer Duplicate",
            "customer_group": "Commercial",
            "territory": "Rest Of The World"
        })
        customer1.insert()
        
        # Attempt to create duplicate
        with self.assertRaises(frappe.DuplicateEntryError):
            customer2 = frappe.get_doc({
                "doctype": "Customer",
                "customer_name": "Test Customer Duplicate",
                "customer_group": "Commercial", 
                "territory": "Rest Of The World"
            })
            customer2.insert()
    
    def test_business_logic_validation(self):
        """Test complex business logic validation"""
        # Create sales order with future delivery date
        so = create_sales_order_for_test({
            "delivery_date": add_days(today(), 30),
            "items": [{
                "item_code": "_Test Item",
                "qty": 10,
                "rate": 100
            }]
        })
        
        # Test delivery date validation
        so.delivery_date = add_days(today(), -1)  # Past date
        
        with self.assertRaises(frappe.ValidationError) as context:
            so.save()
        
        self.assertIn("Delivery Date cannot be in the past", str(context.exception))
```

### 3. Calculation Testing Pattern
```python
class TestCalculations(FrappeTestCase):
    """Test calculation accuracy and edge cases"""
    
    def test_tax_calculation_accuracy(self):
        """Test tax calculation precision"""
        sales_invoice = create_sales_invoice_for_test({
            "items": [{
                "item_code": "_Test Item",
                "qty": 3,
                "rate": 333.33  # Test decimal precision
            }],
            "taxes_and_charges": "_Test Tax Template"
        })
        
        # Validate calculations
        self.assertEqual(sales_invoice.net_total, 999.99)
        self.assertEqual(sales_invoice.total_taxes_and_charges, 180.00)
        self.assertEqual(sales_invoice.grand_total, 1179.99)
    
    def test_currency_conversion(self):
        """Test multi-currency calculations"""
        # Create USD sales invoice
        sales_invoice = create_sales_invoice_for_test({
            "currency": "USD",
            "conversion_rate": 75.0,
            "items": [{
                "item_code": "_Test Item", 
                "qty": 1,
                "rate": 100  # USD
            }]
        })
        
        # Validate conversions
        self.assertEqual(sales_invoice.base_net_total, 7500.0)  # 100 * 75
        self.assertEqual(sales_invoice.net_total, 100.0)
```

## Integration Testing

### 1. Workflow Integration Testing
```python
class TestSalesOrderWorkflow(FrappeTestCase):
    """Test complete sales order workflow integration"""
    
    def test_quotation_to_invoice_workflow(self):
        """Test complete quotation to invoice workflow"""
        
        # Step 1: Create quotation
        quotation = create_quotation_for_test({
            "party_name": "_Test Customer",
            "quotation_to": "Customer",
            "items": [{
                "item_code": "_Test Item",
                "qty": 5,
                "rate": 100
            }]
        })
        
        self.assertEqual(quotation.docstatus, 1)
        self.assertEqual(quotation.status, "Submitted")
        
        # Step 2: Convert to sales order
        from erpnext.selling.doctype.quotation.quotation import make_sales_order
        sales_order = make_sales_order(quotation.name)
        sales_order.delivery_date = add_days(today(), 10)
        sales_order.insert()
        sales_order.submit()
        
        self.assertEqual(sales_order.docstatus, 1)
        self.assertEqual(sales_order.quotation_no, quotation.name)
        
        # Step 3: Create delivery note
        from erpnext.selling.doctype.sales_order.sales_order import make_delivery_note
        delivery_note = make_delivery_note(sales_order.name)
        delivery_note.insert()
        delivery_note.submit()
        
        # Validate stock impact
        stock_balance = get_stock_balance("_Test Item", "_Test Warehouse - _TC")
        self.assertEqual(stock_balance, 95)  # Assuming started with 100
        
        # Step 4: Create sales invoice
        from erpnext.selling.doctype.sales_order.sales_order import make_sales_invoice
        sales_invoice = make_sales_invoice(sales_order.name)
        sales_invoice.insert()
        sales_invoice.submit()
        
        # Validate accounting entries
        gl_entries = frappe.get_all("GL Entry", 
                                   filters={"voucher_no": sales_invoice.name},
                                   fields=["account", "debit", "credit"])
        
        total_debit = sum(entry.debit for entry in gl_entries)
        total_credit = sum(entry.credit for entry in gl_entries)
        
        self.assertEqual(total_debit, total_credit)
        self.assertEqual(total_debit, sales_invoice.grand_total)
```

### 2. Multi-Module Integration
```python
class TestAccountsStockIntegration(FrappeTestCase):
    """Test accounts and stock module integration"""
    
    def test_perpetual_inventory_impact(self):
        """Test perpetual inventory accounting impact"""
        
        # Enable perpetual inventory
        company = frappe.get_doc("Company", "_Test Company")
        company.enable_perpetual_inventory = 1
        company.save()
        
        # Create purchase receipt
        purchase_receipt = create_purchase_receipt_for_test({
            "items": [{
                "item_code": "_Test Item",
                "qty": 100,
                "rate": 50,
                "warehouse": "_Test Warehouse - _TC"
            }]
        })
        
        # Validate stock ledger
        sle = frappe.get_all("Stock Ledger Entry",
                           filters={"voucher_no": purchase_receipt.name},
                           fields=["actual_qty", "stock_value"])
        
        self.assertEqual(len(sle), 1)
        self.assertEqual(sle[0].actual_qty, 100)
        self.assertEqual(sle[0].stock_value, 5000)
        
        # Validate GL entries
        gl_entries = frappe.get_all("GL Entry",
                                  filters={"voucher_no": purchase_receipt.name},
                                  fields=["account", "debit", "credit"])
        
        stock_in_hand_entry = next(
            (entry for entry in gl_entries if "Stock In Hand" in entry.account),
            None
        )
        
        self.assertIsNotNone(stock_in_hand_entry)
        self.assertEqual(stock_in_hand_entry.debit, 5000)
```

### 3. Permission Integration Testing
```python
class TestPermissionIntegration(FrappeTestCase):
    """Test role-based permission integration"""
    
    def setUp(self):
        """Set up users with different roles"""
        self.sales_user = create_test_user("sales@test.com", ["Sales User"])
        self.accounts_user = create_test_user("accounts@test.com", ["Accounts User"])
        self.sales_manager = create_test_user("manager@test.com", ["Sales Manager"])
    
    def test_role_based_document_access(self):
        """Test document access based on user roles"""
        
        # Create sales order as sales user
        frappe.set_user(self.sales_user.name)
        
        sales_order = create_sales_order_for_test({
            "customer": "_Test Customer",
            "items": [{"item_code": "_Test Item", "qty": 1, "rate": 100}]
        })
        
        # Sales user should be able to read their own document
        self.assertTrue(frappe.has_permission("Sales Order", "read", sales_order.name))
        
        # Switch to accounts user
        frappe.set_user(self.accounts_user.name)
        
        # Accounts user should not have write access to sales order
        self.assertFalse(frappe.has_permission("Sales Order", "write", sales_order.name))
        
        # But should have read access for reporting
        self.assertTrue(frappe.has_permission("Sales Order", "read", sales_order.name))
        
        # Sales manager should have full access
        frappe.set_user(self.sales_manager.name)
        self.assertTrue(frappe.has_permission("Sales Order", "write", sales_order.name))
        self.assertTrue(frappe.has_permission("Sales Order", "cancel", sales_order.name))
```

## Performance Testing

### 1. Database Index Testing
```python
# From erpnext/tests/test_perf.py:11
class TestPerformance(FrappeTestCase):
    """Test performance-critical database operations"""
    
    def test_ensure_indexes(self):
        """Ensure critical database indexes exist"""
        INDEXED_FIELDS = {
            "Bin": ["item_code"],
            "GL Entry": ["voucher_no", "posting_date", "company", "party"],
            "Purchase Order Item": ["item_code"],
            "Stock Ledger Entry": ["item_code", "warehouse", "posting_date"],
            "Sales Invoice": ["customer", "posting_date", "company"],
        }
        
        for doctype, fields in INDEXED_FIELDS.items():
            for field in fields:
                indexes = frappe.db.sql(f"""
                    SHOW INDEX FROM `tab{doctype}`
                    WHERE Column_name = "{field}" AND Seq_in_index = 1
                """)
                
                self.assertTrue(
                    len(indexes) > 0,
                    f"Missing index on {doctype}.{field}"
                )
    
    def test_query_performance(self):
        """Test query performance with large datasets"""
        import time
        
        # Create test data
        self.create_performance_test_data()
        
        # Test item search performance
        start_time = time.time()
        
        items = frappe.db.sql("""
            SELECT i.name, i.item_code, i.item_name, 
                   b.actual_qty, ip.price_list_rate
            FROM `tabItem` i
            LEFT JOIN `tabBin` b ON i.item_code = b.item_code
            LEFT JOIN `tabItem Price` ip ON i.item_code = ip.item_code
            WHERE i.item_group = 'Products'
              AND i.disabled = 0
              AND (b.actual_qty > 0 OR i.is_stock_item = 0)
            ORDER BY i.item_name
            LIMIT 100
        """, as_dict=1)
        
        execution_time = time.time() - start_time
        
        # Performance assertion - query should complete within 1 second
        self.assertLess(execution_time, 1.0, 
                       f"Item search query took {execution_time:.2f}s, expected < 1.0s")
        
        self.assertGreater(len(items), 0, "Query should return results")
    
    def create_performance_test_data(self):
        """Create large dataset for performance testing"""
        if frappe.db.exists("Item", "_Test Perf Item 1000"):
            return  # Data already exists
        
        # Create 1000 test items
        for i in range(1, 1001):
            item = frappe.get_doc({
                "doctype": "Item",
                "item_code": f"_Test Perf Item {i}",
                "item_name": f"Test Performance Item {i}",
                "item_group": "Products",
                "stock_uom": "Nos",
                "is_stock_item": 1
            })
            item.insert(ignore_permissions=True)
            
            # Create stock entry for every 10th item
            if i % 10 == 0:
                create_stock_entry_for_test({
                    "item_code": item.item_code,
                    "qty": 100,
                    "warehouse": "_Test Warehouse - _TC"
                })
```

### 2. Memory Usage Testing
```python
class TestMemoryUsage(FrappeTestCase):
    """Test memory usage patterns"""
    
    def test_large_report_memory(self):
        """Test memory usage for large reports"""
        import psutil
        import gc
        
        process = psutil.Process()
        
        # Measure initial memory
        initial_memory = process.memory_info().rss / 1024 / 1024  # MB
        
        # Generate large report
        filters = {
            "company": "_Test Company",
            "from_date": "2020-01-01",
            "to_date": "2023-12-31"
        }
        
        from erpnext.selling.report.sales_analytics.sales_analytics import execute
        columns, data = execute(filters)
        
        # Measure peak memory
        peak_memory = process.memory_info().rss / 1024 / 1024  # MB
        memory_increase = peak_memory - initial_memory
        
        # Memory increase should be reasonable (< 500MB for large reports)
        self.assertLess(memory_increase, 500, 
                       f"Report used {memory_increase:.2f}MB, expected < 500MB")
        
        # Clean up
        del data
        gc.collect()
```

### 3. Load Testing Simulation
```python
class TestLoadScenarios(FrappeTestCase):
    """Test system behavior under load"""
    
    def test_concurrent_sales_invoice_creation(self):
        """Test concurrent document creation"""
        import threading
        import queue
        
        results = queue.Queue()
        errors = queue.Queue()
        
        def create_invoice(thread_id):
            """Create sales invoice in thread"""
            try:
                frappe.local.response = frappe._dict({'type': None})
                frappe.local.response_headers = []
                
                invoice = create_sales_invoice_for_test({
                    "customer": "_Test Customer",
                    "items": [{
                        "item_code": "_Test Item",
                        "qty": 1,
                        "rate": 100
                    }]
                })
                
                results.put((thread_id, invoice.name))
                
            except Exception as e:
                errors.put((thread_id, str(e)))
        
        # Create 10 concurrent threads
        threads = []
        for i in range(10):
            thread = threading.Thread(target=create_invoice, args=(i,))
            threads.append(thread)
            thread.start()
        
        # Wait for completion
        for thread in threads:
            thread.join()
        
        # Validate results
        successful_creates = results.qsize()
        failed_creates = errors.qsize()
        
        self.assertGreater(successful_creates, 7,  # Allow some failures
                          f"Only {successful_creates}/10 invoices created successfully")
        
        # Check for database consistency
        if not errors.empty():
            while not errors.empty():
                thread_id, error = errors.get()
                print(f"Thread {thread_id} failed: {error}")
```

## API Testing

### 1. REST API Testing Pattern
```python
class TestRESTAPI(FrappeTestCase):
    """Test REST API endpoints"""
    
    def setUp(self):
        """Set up API testing environment"""
        self.client = frappe.client.FrappeClient(
            frappe.get_site_path(),
            username="Administrator",
            password="admin"
        )
    
    def test_get_document_api(self):
        """Test document retrieval via REST API"""
        # Create test customer
        customer = create_customer_for_test("_Test API Customer")
        
        # Test GET request
        response = self.client.get_doc("Customer", customer.name)
        
        self.assertEqual(response["name"], customer.name)
        self.assertEqual(response["customer_name"], customer.customer_name)
        self.assertEqual(response["doctype"], "Customer")
    
    def test_create_document_api(self):
        """Test document creation via REST API"""
        customer_data = {
            "doctype": "Customer",
            "customer_name": "_Test API Create Customer",
            "customer_group": "Commercial",
            "territory": "Rest Of The World"
        }
        
        response = self.client.insert(customer_data)
        
        self.assertEqual(response["customer_name"], customer_data["customer_name"])
        self.assertTrue(frappe.db.exists("Customer", response["name"]))
    
    def test_api_error_handling(self):
        """Test API error responses"""
        with self.assertRaises(frappe.DoesNotExistError):
            self.client.get_doc("Customer", "Non Existent Customer")
    
    def test_api_permissions(self):
        """Test API permission enforcement"""
        # Create user with limited permissions
        limited_user = create_test_user("limited@test.com", ["Sales User"])
        
        limited_client = frappe.client.FrappeClient(
            frappe.get_site_path(),
            username="limited@test.com",
            password="test123"
        )
        
        # Test should fail for restricted access
        with self.assertRaises(frappe.PermissionError):
            limited_client.get_list("Company")
```

### 2. Webhook Testing
```python
class TestWebhooks(FrappeTestCase):
    """Test webhook integrations"""
    
    def test_payment_webhook_processing(self):
        """Test payment gateway webhook processing"""
        import json
        import hmac
        import hashlib
        
        # Simulate webhook payload
        webhook_data = {
            "event_type": "payment_success",
            "payment_id": "pay_test_123",
            "order_id": "order_test_456",
            "amount": 1000,
            "currency": "INR",
            "status": "captured"
        }
        
        # Generate webhook signature
        secret = "test_webhook_secret"
        signature = hmac.new(
            secret.encode(),
            json.dumps(webhook_data).encode(),
            hashlib.sha256
        ).hexdigest()
        
        # Process webhook
        from myapp.api import process_payment_webhook
        result = process_payment_webhook(
            signature=signature,
            data=json.dumps(webhook_data)
        )
        
        self.assertTrue(result["success"])
        
        # Validate payment record created
        payment_entry = frappe.get_all("Payment Entry",
                                     filters={"reference_no": "pay_test_123"},
                                     limit=1)
        
        self.assertEqual(len(payment_entry), 1)
```

## Report Testing

### 1. Report Data Accuracy Testing
```python
# From erpnext/selling/report/sales_analytics/test_analytics.py
class TestSalesAnalytics(FrappeTestCase):
    """Test sales analytics report accuracy"""
    
    def setUp(self):
        """Set up report test data"""
        self.create_test_sales_data()
    
    def create_test_sales_data(self):
        """Create consistent test data for reports"""
        # Create sales invoices for testing
        for month in range(1, 4):  # Jan, Feb, Mar
            for day in [1, 15]:  # Two invoices per month
                invoice = create_sales_invoice_for_test({
                    "customer": "_Test Customer",
                    "posting_date": f"2023-{month:02d}-{day:02d}",
                    "items": [{
                        "item_code": "_Test Item",
                        "qty": 10,
                        "rate": 100
                    }]
                })
    
    def test_monthly_sales_analytics(self):
        """Test monthly sales analytics calculations"""
        from erpnext.selling.report.sales_analytics.sales_analytics import execute
        
        filters = {
            "doc_type": "Sales Invoice",
            "value_quantity": "Value",
            "tree_type": "Customer",
            "based_on": "Item",
            "from_date": "2023-01-01",
            "to_date": "2023-03-31",
            "company": "_Test Company",
            "range": "Monthly"
        }
        
        columns, data = execute(filters)
        
        # Validate column structure
        expected_columns = ["Item", "Jan 2023", "Feb 2023", "Mar 2023", "Total"]
        actual_column_labels = [col.get("label") or col for col in columns]
        
        for expected_col in expected_columns:
            self.assertIn(expected_col, actual_column_labels)
        
        # Validate data accuracy
        test_item_row = next((row for row in data if row[0] == "_Test Item"), None)
        self.assertIsNotNone(test_item_row, "Test item not found in report data")
        
        # Each month should have 2000 (2 invoices × 10 qty × 100 rate)
        self.assertEqual(test_item_row[1], 2000)  # Jan 2023
        self.assertEqual(test_item_row[2], 2000)  # Feb 2023
        self.assertEqual(test_item_row[3], 2000)  # Mar 2023
        self.assertEqual(test_item_row[4], 6000)  # Total
    
    def test_customer_wise_analytics(self):
        """Test customer-wise analytics grouping"""
        filters = {
            "doc_type": "Sales Invoice",
            "tree_type": "Customer",
            "based_on": "Customer",
            "from_date": "2023-01-01",
            "to_date": "2023-03-31"
        }
        
        columns, data = execute(filters)
        
        # Validate customer grouping
        customer_names = [row[0] for row in data if not row[0].startswith(" ")]
        self.assertIn("_Test Customer", customer_names)
```

### 2. Report Performance Testing
```python
class TestReportPerformance(FrappeTestCase):
    """Test report generation performance"""
    
    def test_large_dataset_report_performance(self):
        """Test report performance with large datasets"""
        import time
        
        # Create large dataset
        self.create_large_test_dataset()
        
        # Test report generation time
        start_time = time.time()
        
        from erpnext.accounts.report.general_ledger.general_ledger import execute
        filters = {
            "company": "_Test Company",
            "from_date": "2020-01-01",
            "to_date": "2023-12-31",
            "include_default_book_entries": 1
        }
        
        columns, data = execute(filters)
        
        execution_time = time.time() - start_time
        
        # Report should complete within reasonable time
        self.assertLess(execution_time, 30.0,  # 30 seconds max
                       f"Report took {execution_time:.2f}s, expected < 30s")
        
        self.assertGreater(len(data), 0, "Report should return data")
    
    def create_large_test_dataset(self):
        """Create large dataset for performance testing"""
        # Create test data only if not exists
        if frappe.db.exists("Sales Invoice", "_Test Perf Invoice 100"):
            return
        
        # Create 100 sales invoices with multiple items each
        for i in range(1, 101):
            invoice = create_sales_invoice_for_test({
                "customer": "_Test Customer",
                "posting_date": f"2022-{(i % 12) + 1:02d}-{(i % 28) + 1:02d}",
                "items": [
                    {"item_code": "_Test Item", "qty": 10, "rate": 100},
                    {"item_code": "_Test Item 2", "qty": 5, "rate": 200},
                ]
            })
            invoice.name = f"_Test Perf Invoice {i}"
            invoice.save()
```

## Security Testing

### 1. Permission Testing
```python
class TestSecurityPermissions(FrappeTestCase):
    """Test security and permission enforcement"""
    
    def setUp(self):
        """Set up security test users"""
        self.guest_user = "Guest"
        self.normal_user = create_test_user("normal@test.com", ["Employee"])
        self.manager_user = create_test_user("manager@test.com", ["Sales Manager"])
    
    def test_guest_user_restrictions(self):
        """Test guest user access restrictions"""
        frappe.set_user(self.guest_user)
        
        # Guest should not access internal documents
        with self.assertRaises(frappe.PermissionError):
            frappe.get_doc("Sales Invoice", "_Test Sales Invoice")
        
        # Guest should not create documents
        with self.assertRaises(frappe.PermissionError):
            customer = frappe.get_doc({
                "doctype": "Customer",
                "customer_name": "Unauthorized Customer"
            })
            customer.insert()
    
    def test_field_level_permissions(self):
        """Test field-level permission enforcement"""
        frappe.set_user(self.normal_user.name)
        
        # Create sales invoice
        invoice = create_sales_invoice_for_test({
            "customer": "_Test Customer",
            "items": [{"item_code": "_Test Item", "qty": 1, "rate": 100}]
        })
        
        # Normal user should not see confidential fields
        invoice_dict = invoice.as_dict()
        self.assertNotIn("confidential_notes", invoice_dict)
        
        # Manager should see all fields
        frappe.set_user(self.manager_user.name)
        invoice_dict = frappe.get_doc("Sales Invoice", invoice.name).as_dict()
        # Manager-specific fields should be visible
    
    def test_row_level_security(self):
        """Test row-level security based on user restrictions"""
        # Create user with specific company access
        restricted_user = create_test_user("restricted@test.com", ["Sales User"])
        add_user_company_restriction(restricted_user.name, "_Test Company 1")
        
        frappe.set_user(restricted_user.name)
        
        # User should only see documents from their company
        sales_orders = frappe.get_all("Sales Order", fields=["name", "company"])
        
        for order in sales_orders:
            self.assertEqual(order.company, "_Test Company 1",
                           f"User saw order from unauthorized company: {order.company}")
```

### 2. Input Validation Testing
```python
class TestInputValidation(FrappeTestCase):
    """Test input validation and sanitization"""
    
    def test_xss_prevention(self):
        """Test XSS attack prevention"""
        malicious_input = "<script>alert('xss')</script>"
        
        customer = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": malicious_input,
            "customer_group": "Commercial",
            "territory": "Rest Of The World"
        })
        customer.insert()
        
        # Customer name should be sanitized
        saved_customer = frappe.get_doc("Customer", customer.name)
        self.assertNotIn("<script>", saved_customer.customer_name)
        self.assertNotIn("alert", saved_customer.customer_name)
    
    def test_sql_injection_prevention(self):
        """Test SQL injection prevention"""
        malicious_filter = "'; DROP TABLE tabCustomer; --"
        
        # This should not cause SQL injection
        try:
            customers = frappe.get_all("Customer", 
                                     filters={"customer_name": malicious_filter})
            # Should return empty result, not cause database error
            self.assertEqual(len(customers), 0)
        except Exception as e:
            # Should not be a database error
            self.assertNotIn("DROP TABLE", str(e))
    
    def test_file_upload_validation(self):
        """Test file upload security validation"""
        # Test malicious file upload
        malicious_file_content = b"<?php system($_GET['cmd']); ?>"
        
        with self.assertRaises(frappe.ValidationError):
            file_doc = frappe.get_doc({
                "doctype": "File",
                "file_name": "malicious.php",
                "content": malicious_file_content
            })
            file_doc.insert()
```

## UI/UX Testing

### 1. Form Validation Testing
```python
class TestFormValidation(FrappeTestCase):
    """Test client-side form validation"""
    
    def test_mandatory_field_validation(self):
        """Test mandatory field validation in forms"""
        # This would typically be tested with Selenium/browser automation
        # Here we test the server-side validation that supports UI validation
        
        with self.assertRaises(frappe.ValidationError) as context:
            customer = frappe.get_doc({
                "doctype": "Customer",
                # Missing mandatory customer_name
                "customer_group": "Commercial",
                "territory": "Rest Of The World"
            })
            customer.insert()
        
        self.assertIn("customer_name", str(context.exception).lower())
    
    def test_custom_validation_messages(self):
        """Test custom validation message display"""
        with self.assertRaises(frappe.ValidationError) as context:
            # Create sales order with past delivery date
            so = frappe.get_doc({
                "doctype": "Sales Order",
                "customer": "_Test Customer",
                "delivery_date": "2020-01-01",  # Past date
                "items": [{
                    "item_code": "_Test Item",
                    "qty": 1,
                    "rate": 100
                }]
            })
            so.insert()
        
        # Should have user-friendly validation message
        error_message = str(context.exception)
        self.assertIn("Delivery Date", error_message)
        self.assertTrue(any(word in error_message.lower() 
                          for word in ["past", "future", "invalid"]))
```

### 2. User Experience Testing  
```python
class TestUserExperience(FrappeTestCase):
    """Test user experience patterns"""
    
    def test_default_value_setting(self):
        """Test automatic default value setting"""
        # Create sales order - should auto-set company, currency, etc.
        so = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": "_Test Customer",
            "items": [{
                "item_code": "_Test Item",
                "qty": 1
            }]
        })
        
        # Default values should be set before validation
        so.set_missing_values()
        
        self.assertIsNotNone(so.company)
        self.assertIsNotNone(so.currency)
        self.assertIsNotNone(so.price_list)
        self.assertGreater(so.items[0].rate, 0)  # Rate should be fetched
    
    def test_auto_calculation_accuracy(self):
        """Test automatic calculation accuracy"""
        so = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": "_Test Customer",
            "items": [
                {"item_code": "_Test Item", "qty": 2, "rate": 100},
                {"item_code": "_Test Item 2", "qty": 3, "rate": 150}
            ]
        })
        
        so.calculate_taxes_and_totals()
        
        # Test calculations
        self.assertEqual(so.items[0].amount, 200)  # 2 * 100
        self.assertEqual(so.items[1].amount, 450)  # 3 * 150
        self.assertEqual(so.total, 650)  # 200 + 450
```

## Test Data Management

### 1. Test Data Creation Patterns
```python
class TestDataFactory:
    """Factory for creating consistent test data"""
    
    @staticmethod
    def create_customer(customer_name=None, **kwargs):
        """Create test customer with defaults"""
        if not customer_name:
            customer_name = f"_Test Customer {frappe.generate_hash()[:8]}"
        
        customer_data = {
            "doctype": "Customer",
            "customer_name": customer_name,
            "customer_group": "Commercial",
            "territory": "Rest Of The World",
            "customer_type": "Company"
        }
        customer_data.update(kwargs)
        
        customer = frappe.get_doc(customer_data)
        customer.insert(ignore_permissions=True)
        return customer
    
    @staticmethod
    def create_item(item_code=None, **kwargs):
        """Create test item with defaults"""
        if not item_code:
            item_code = f"_Test Item {frappe.generate_hash()[:8]}"
        
        item_data = {
            "doctype": "Item",
            "item_code": item_code,
            "item_name": item_code,
            "item_group": "All Item Groups",
            "stock_uom": "Nos",
            "is_stock_item": 1,
            "valuation_rate": 100
        }
        item_data.update(kwargs)
        
        item = frappe.get_doc(item_data)
        item.insert(ignore_permissions=True)
        return item
    
    @staticmethod
    def create_sales_order_with_items(customer=None, items_data=None):
        """Create sales order with items"""
        if not customer:
            customer = TestDataFactory.create_customer().name
        
        if not items_data:
            items_data = [{"item_code": "_Test Item", "qty": 1, "rate": 100}]
        
        so = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": customer,
            "delivery_date": add_days(today(), 7),
            "items": items_data
        })
        so.insert(ignore_permissions=True)
        so.submit()
        return so
```

### 2. Test Data Cleanup Patterns
```python
class TestDataCleaner:
    """Centralized test data cleanup"""
    
    def __init__(self):
        self.created_documents = []
    
    def track_document(self, doctype, name):
        """Track document for cleanup"""
        self.created_documents.append((doctype, name))
    
    def cleanup_all(self):
        """Clean up all tracked documents"""
        for doctype, name in reversed(self.created_documents):
            try:
                if frappe.db.exists(doctype, name):
                    doc = frappe.get_doc(doctype, name)
                    if doc.docstatus == 1:
                        doc.cancel()
                    doc.delete()
            except Exception as e:
                print(f"Failed to cleanup {doctype} {name}: {e}")
        
        self.created_documents.clear()

# Usage in test cases
class TestWithDataCleanup(FrappeTestCase):
    def setUp(self):
        self.cleaner = TestDataCleaner()
    
    def tearDown(self):
        self.cleaner.cleanup_all()
    
    def test_with_cleanup(self):
        customer = TestDataFactory.create_customer()
        self.cleaner.track_document("Customer", customer.name)
        
        # Test logic here
        pass
```

### 3. Database Transaction Management
```python
class TestTransactionManagement(FrappeTestCase):
    """Test database transaction handling"""
    
    @classmethod
    def setUpClass(cls):
        """Set up class-level savepoint"""
        frappe.db.savepoint("test_class_savepoint")
        cls.setup_class_data()
    
    @classmethod
    def tearDownClass(cls):
        """Rollback to class savepoint"""
        frappe.db.rollback(save_point="test_class_savepoint")
    
    def setUp(self):
        """Set up test-level savepoint"""
        frappe.db.savepoint("test_method_savepoint")
    
    def tearDown(self):
        """Rollback to test savepoint"""
        frappe.db.rollback(save_point="test_method_savepoint")
```

## Continuous Testing

### 1. CI/CD Integration Testing
```python
# .github/workflows/test.yml equivalent in code
class TestCIIntegration:
    """Test CI/CD integration patterns"""
    
    def setup_test_environment(self):
        """Set up consistent test environment"""
        # Reset database to known state
        frappe.db.sql("DELETE FROM `tabCustomer` WHERE name LIKE '_Test%'")
        frappe.db.sql("DELETE FROM `tabItem` WHERE name LIKE '_Test%'")
        
        # Create base test data
        self.create_base_test_data()
        
        # Set test-specific settings
        frappe.db.set_single_value("System Settings", "country", "India")
        frappe.db.set_single_value("Selling Settings", "cust_master_name", "Customer Name")
    
    def run_test_suite(self):
        """Run complete test suite with proper ordering"""
        test_modules = [
            "erpnext.tests.test_init",
            "erpnext.accounts.doctype.account.test_account",
            "erpnext.stock.doctype.item.test_item",
            "erpnext.selling.doctype.customer.test_customer",
            "erpnext.tests.test_point_of_sale"
        ]
        
        results = {}
        
        for module in test_modules:
            try:
                # Import and run tests
                test_module = frappe.get_module(module)
                suite = unittest.TestLoader().loadTestsFromModule(test_module)
                runner = unittest.TextTestRunner(verbosity=2)
                result = runner.run(suite)
                
                results[module] = {
                    "success": result.wasSuccessful(),
                    "tests_run": result.testsRun,
                    "failures": len(result.failures),
                    "errors": len(result.errors)
                }
                
            except Exception as e:
                results[module] = {"error": str(e)}
        
        return results
```

### 2. Test Coverage Analysis
```python
class TestCoverageAnalysis:
    """Analyze and ensure adequate test coverage"""
    
    def analyze_module_coverage(self, module_path):
        """Analyze test coverage for a module"""
        import coverage
        
        cov = coverage.Coverage()
        cov.start()
        
        # Import and run module tests
        try:
            test_module = frappe.get_module(f"{module_path}.test_{module_path.split('.')[-1]}")
            suite = unittest.TestLoader().loadTestsFromModule(test_module)
            runner = unittest.TextTestRunner(stream=open(os.devnull, 'w'))
            runner.run(suite)
        except ImportError:
            pass
        
        cov.stop()
        cov.save()
        
        # Generate coverage report
        coverage_data = cov.get_data()
        
        return {
            "files_covered": len(coverage_data.measured_files()),
            "statements_covered": coverage_data.line_count(),
            "coverage_percentage": cov.report(show_missing=False)
        }
    
    def ensure_minimum_coverage(self, minimum_percentage=80):
        """Ensure minimum test coverage threshold"""
        core_modules = [
            "erpnext.accounts",
            "erpnext.stock", 
            "erpnext.selling",
            "erpnext.buying"
        ]
        
        for module in core_modules:
            coverage_data = self.analyze_module_coverage(module)
            
            if coverage_data["coverage_percentage"] < minimum_percentage:
                raise AssertionError(
                    f"Module {module} has {coverage_data['coverage_percentage']}% coverage, "
                    f"minimum required is {minimum_percentage}%"
                )
```

## Testing Best Practices

### 1. Test Organization Best Practices
```python
"""
Best Practices for Test Organization:

1. File Naming:
   - test_[module_name].py for unit tests
   - test_[functionality].py for integration tests
   - Use descriptive names that match the code being tested

2. Test Class Structure:
   - One test class per major component
   - Group related tests within the same class
   - Use descriptive class and method names

3. Test Method Naming:
   - Start with 'test_'
   - Use descriptive names: test_calculate_tax_with_discount()
   - Include the expected outcome: test_create_customer_success()

4. Documentation:
   - Add docstrings to all test classes and methods
   - Explain the business scenario being tested
   - Document expected vs actual behavior
"""

class TestBestPracticesExample(FrappeTestCase):
    """
    Example of well-organized test class
    
    Tests customer creation, validation, and business logic
    following ERPNext testing best practices.
    """
    
    def setUp(self):
        """Set up test environment before each test method"""
        self.test_company = "_Test Company"
        self.test_territory = "Rest Of The World"
        self.cleanup_test_data()
    
    def tearDown(self):
        """Clean up after each test method"""
        self.cleanup_test_data()
    
    def cleanup_test_data(self):
        """Remove any test data that might interfere"""
        test_customers = frappe.get_all("Customer", 
                                       filters={"customer_name": ["like", "_Test%"]})
        for customer in test_customers:
            frappe.delete_doc("Customer", customer.name, ignore_permissions=True)
    
    def test_create_customer_with_required_fields_success(self):
        """Test successful customer creation with minimum required fields"""
        customer = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": "_Test Customer Required Fields",
            "customer_group": "Commercial",
            "territory": self.test_territory
        })
        
        # Should create successfully
        customer.insert()
        
        # Verify creation
        self.assertTrue(frappe.db.exists("Customer", customer.name))
        self.assertEqual(customer.customer_name, "_Test Customer Required Fields")
    
    def test_create_customer_missing_required_field_fails(self):
        """Test customer creation fails when required fields are missing"""
        customer = frappe.get_doc({
            "doctype": "Customer",
            # Missing customer_name (required field)
            "customer_group": "Commercial",
            "territory": self.test_territory
        })
        
        # Should raise validation error
        with self.assertRaises(frappe.ValidationError) as context:
            customer.insert()
        
        # Verify error message mentions missing field
        self.assertIn("customer_name", str(context.exception).lower())
```

### 2. Assertion Best Practices
```python
class TestAssertionBestPractices(FrappeTestCase):
    """Examples of effective test assertions"""
    
    def test_assertions_with_clear_messages(self):
        """Use clear, descriptive assertion messages"""
        invoice = create_sales_invoice_for_test()
        
        # Good: Descriptive assertion message
        self.assertEqual(invoice.status, "Draft", 
                        f"New invoice should have Draft status, got {invoice.status}")
        
        # Good: Multiple related assertions grouped together
        self.assertIsNotNone(invoice.grand_total, "Grand total should be calculated")
        self.assertGreater(invoice.grand_total, 0, "Grand total should be positive")
        self.assertEqual(invoice.currency, "INR", "Default currency should be INR")
    
    def test_floating_point_comparisons(self):
        """Proper handling of floating point comparisons"""
        invoice = create_sales_invoice_for_test({
            "items": [{"item_code": "_Test Item", "qty": 3, "rate": 333.33}]
        })
        
        expected_total = 999.99
        
        # Good: Use assertAlmostEqual for floating point comparisons
        self.assertAlmostEqual(invoice.grand_total, expected_total, places=2,
                             msg=f"Grand total calculation incorrect: "
                                 f"expected {expected_total}, got {invoice.grand_total}")
    
    def test_collection_assertions(self):
        """Effective testing of collections and lists"""
        customer = create_customer_for_test()
        
        # Create multiple addresses
        for i in range(3):
            address = frappe.get_doc({
                "doctype": "Address",
                "address_title": f"Test Address {i}",
                "address_line1": f"Line 1 - {i}",
                "city": "Test City",
                "links": [{"link_doctype": "Customer", "link_name": customer.name}]
            })
            address.insert()
        
        addresses = frappe.get_all("Address", 
                                  filters={"address_title": ["like", "Test Address%"]})
        
        # Good: Test collection properties
        self.assertEqual(len(addresses), 3, "Should have created 3 addresses")
        
        # Good: Test specific items in collection
        address_titles = [addr.address_title for addr in addresses]
        self.assertIn("Test Address 0", address_titles, 
                     "First test address should exist")
        
        # Good: Test all items meet criteria
        for address in addresses:
            self.assertTrue(address.address_title.startswith("Test Address"),
                          f"Address title '{address.address_title}' should start with 'Test Address'")
```

### 3. Test Data Best Practices
```python
class TestDataBestPractices(FrappeTestCase):
    """Best practices for test data management"""
    
    def test_use_meaningful_test_data(self):
        """Use realistic, meaningful test data"""
        # Good: Realistic test data that represents actual use cases
        customer = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": "Acme Corporation",
            "customer_type": "Company",
            "customer_group": "Commercial",
            "territory": "United States",
            "default_currency": "USD"
        })
        customer.insert()
        
        # Test with realistic business scenario
        sales_order = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": customer.name,
            "transaction_date": today(),
            "delivery_date": add_days(today(), 14),  # 2 weeks delivery
            "items": [{
                "item_code": "_Test Item",
                "item_name": "Test Product",
                "qty": 100,  # Realistic quantity
                "rate": 25.50,  # Realistic price
                "uom": "Nos"
            }]
        })
        sales_order.insert()
        
        # Verify realistic calculations
        expected_amount = 100 * 25.50
        self.assertEqual(sales_order.items[0].amount, expected_amount)
    
    def test_boundary_value_testing(self):
        """Test boundary values and edge cases"""
        # Test minimum values
        item_min = frappe.get_doc({
            "doctype": "Item",
            "item_code": "_Test Item Min",
            "item_name": "Test Item Minimum",
            "item_group": "All Item Groups",
            "stock_uom": "Nos",
            "min_order_qty": 1  # Minimum possible
        })
        item_min.insert()
        
        # Test maximum values (within system limits)
        item_max = frappe.get_doc({
            "doctype": "Item", 
            "item_code": "_Test Item Max",
            "item_name": "Test Item Maximum",
            "item_group": "All Item Groups",
            "stock_uom": "Nos",
            "max_discount": 100  # Maximum discount
        })
        item_max.insert()
        
        # Test zero values where applicable
        invoice_zero = create_sales_invoice_for_test({
            "items": [{
                "item_code": "_Test Item",
                "qty": 1,
                "rate": 0  # Zero rate edge case
            }]
        })
        
        self.assertEqual(invoice_zero.grand_total, 0)
```

## Debugging & Troubleshooting

### 1. Test Debugging Techniques
```python
class TestDebugging(FrappeTestCase):
    """Techniques for debugging failing tests"""
    
    def test_with_debug_information(self):
        """Include debug information in test failures"""
        customer = create_customer_for_test("Debug Test Customer")
        
        # Create sales order with debug context
        try:
            sales_order = frappe.get_doc({
                "doctype": "Sales Order",
                "customer": customer.name,
                "items": [{
                    "item_code": "_Test Item",
                    "qty": 10,
                    "rate": 100
                }]
            })
            sales_order.insert()
            
            # Add debug information to assertions
            self.assertEqual(sales_order.status, "Draft",
                           f"Sales Order Debug Info:\n"
                           f"Name: {sales_order.name}\n"
                           f"Customer: {sales_order.customer}\n"
                           f"Items: {[item.as_dict() for item in sales_order.items]}\n"
                           f"Grand Total: {sales_order.grand_total}")
            
        except Exception as e:
            # Enhanced error reporting with context
            error_context = {
                "customer": customer.as_dict() if customer else None,
                "sales_order_data": sales_order.as_dict() if 'sales_order' in locals() else None,
                "database_state": self.get_debug_database_state(),
                "user": frappe.session.user,
                "site": frappe.local.site
            }
            
            print(f"Test failed with context: {json.dumps(error_context, indent=2, default=str)}")
            raise
    
    def get_debug_database_state(self):
        """Get current database state for debugging"""
        return {
            "customers_count": frappe.db.count("Customer"),
            "items_count": frappe.db.count("Item"),
            "test_data_exists": frappe.db.exists("Customer", "_Test Customer")
        }
    
    def test_with_step_by_step_validation(self):
        """Break down complex tests into validatable steps"""
        # Step 1: Create customer
        customer = create_customer_for_test("Step Test Customer")
        self.assertTrue(frappe.db.exists("Customer", customer.name),
                       "Step 1 failed: Customer not created")
        
        # Step 2: Create item
        item = create_item_for_test("_Test Step Item")
        self.assertTrue(frappe.db.exists("Item", item.name),
                       "Step 2 failed: Item not created")
        
        # Step 3: Create sales order
        so = frappe.get_doc({
            "doctype": "Sales Order",
            "customer": customer.name,
            "items": [{"item_code": item.name, "qty": 1, "rate": 100}]
        })
        so.insert()
        self.assertEqual(so.docstatus, 0,
                       "Step 3 failed: Sales Order not in draft state")
        
        # Step 4: Submit sales order
        so.submit()
        self.assertEqual(so.docstatus, 1,
                       "Step 4 failed: Sales Order not submitted")
```

### 2. Performance Debugging
```python
class TestPerformanceDebugging(FrappeTestCase):
    """Debug performance issues in tests"""
    
    def test_with_timing_information(self):
        """Include timing information for performance analysis"""
        import time
        
        start_time = time.time()
        
        # Create large dataset
        customers = []
        for i in range(100):
            customer = create_customer_for_test(f"Perf Customer {i}")
            customers.append(customer)
        
        creation_time = time.time() - start_time
        
        # Query performance test
        query_start = time.time()
        
        customer_list = frappe.get_all("Customer",
                                     filters={"customer_name": ["like", "Perf Customer%"]},
                                     fields=["name", "customer_name", "creation"])
        
        query_time = time.time() - query_start
        
        # Performance assertions with timing info
        self.assertLess(creation_time, 10.0,
                       f"Customer creation took {creation_time:.2f}s, expected < 10s")
        
        self.assertLess(query_time, 1.0,
                       f"Query took {query_time:.2f}s, expected < 1s")
        
        self.assertEqual(len(customer_list), 100,
                        f"Query returned {len(customer_list)} customers, expected 100")
    
    def test_memory_usage_debugging(self):
        """Debug memory usage issues"""
        import psutil
        import gc
        
        process = psutil.Process()
        initial_memory = process.memory_info().rss / 1024 / 1024  # MB
        
        # Create large data structure
        large_list = []
        for i in range(10000):
            large_list.append(frappe.get_doc({
                "doctype": "Customer",
                "customer_name": f"Memory Test {i}",
                "customer_group": "Commercial",
                "territory": "Rest Of The World"
            }))
        
        peak_memory = process.memory_info().rss / 1024 / 1024  # MB
        memory_increase = peak_memory - initial_memory
        
        # Clean up
        del large_list
        gc.collect()
        
        final_memory = process.memory_info().rss / 1024 / 1024  # MB
        memory_released = peak_memory - final_memory
        
        # Memory usage assertions
        self.assertLess(memory_increase, 500,
                       f"Memory usage increased by {memory_increase:.2f}MB, expected < 500MB")
        
        self.assertGreater(memory_released, memory_increase * 0.8,
                          f"Only {memory_released:.2f}MB released out of {memory_increase:.2f}MB used")
```

---

## Summary

This comprehensive testing guide provides production-ready methodologies extracted from ERPNext's battle-tested codebase. Key takeaways:

1. **Structured Testing**: Organized test hierarchies with clear setup/teardown patterns
2. **Comprehensive Coverage**: Unit, integration, performance, security, and UI testing
3. **Data Management**: Proper test data creation, cleanup, and transaction handling
4. **Best Practices**: Clear assertions, meaningful test data, and effective debugging
5. **Automation Ready**: CI/CD integration patterns and coverage analysis

Use these patterns to build robust, maintainable test suites that ensure quality and reliability in your custom Frappe applications. Each pattern has been proven in ERPNext's production environment and provides excellent foundation for enterprise-grade testing strategies.

**File References:**
- Unit Testing: `erpnext/tests/test_point_of_sale.py:14`
- Integration Testing: `erpnext/support/doctype/service_level_agreement/test_service_level_agreement.py:16`
- Performance Testing: `erpnext/tests/test_perf.py:11`
- Report Testing: `erpnext/selling/report/sales_analytics/test_analytics.py`