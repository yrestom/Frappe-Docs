# Test Templates - Production-Ready Testing Patterns

*Based on ERPNext testing framework analysis - Battle-tested patterns for comprehensive test coverage*

## Table of Contents

1. [Unit Test Templates](#unit-test-templates)
2. [Integration Test Templates](#integration-test-templates)
3. [API Test Templates](#api-test-templates)
4. [Test Data Management](#test-data-management)
5. [Mock and Fixture Patterns](#mock-and-fixture-patterns)
6. [Performance Test Templates](#performance-test-templates)

---

## Unit Test Templates

### DocType Unit Test Template
*Based on ERPNext's Customer and Sales Invoice test patterns*

```python
# test_your_doctype.py
import frappe
import unittest
from frappe.utils import today, add_days, flt, getdate
from your_app.your_module.doctype.your_doctype.your_doctype import YourDocType

class TestYourDocType(unittest.TestCase):
    """
    Comprehensive unit testing following ERPNext patterns
    """
    
    @classmethod
    def setUpClass(cls):
        """Set up test data that's reused across test methods"""
        cls.company = "_Test Company"
        cls.customer = "_Test Customer"
        cls.item = "_Test Item"
        cls.warehouse = "_Test Warehouse"
        
        # Create test dependencies
        cls.setup_test_dependencies()
    
    @classmethod
    def tearDownClass(cls):
        """Clean up after all tests"""
        pass
    
    def setUp(self):
        """Set up before each test method"""
        frappe.db.rollback()
        frappe.clear_cache()
    
    def tearDown(self):
        """Clean up after each test method"""
        frappe.db.rollback()
    
    def test_document_creation(self):
        """Test basic document creation and validation"""
        doc = self.create_test_document()
        
        # Test document creation
        self.assertIsNotNone(doc.name)
        self.assertEqual(doc.doctype, "Your DocType")
        self.assertEqual(doc.customer, self.customer)
        
        # Test auto-set fields
        self.assertEqual(doc.company, self.company)
        self.assertEqual(doc.posting_date, today())
    
    def test_field_validation(self):
        """Test field-level validations"""
        # Test required field validation
        with self.assertRaises(frappe.ValidationError):
            doc = frappe.get_doc({
                "doctype": "Your DocType",
                # Missing required fields
            })
            doc.insert()
        
        # Test invalid data validation
        with self.assertRaises(frappe.ValidationError):
            doc = self.create_test_document()
            doc.start_date = "2023-01-01"
            doc.end_date = "2022-01-01"  # End date before start date
            doc.save()
    
    def test_business_logic_validation(self):
        """Test business rule validations"""
        doc = self.create_test_document()
        
        # Test negative amount validation
        with self.assertRaises(frappe.ValidationError):
            doc.amount = -100
            doc.save()
        
        # Test duplicate prevention
        doc.save()
        
        with self.assertRaises(frappe.DuplicateEntryError):
            duplicate_doc = self.create_test_document()
            duplicate_doc.reference_no = doc.reference_no  # Duplicate reference
            duplicate_doc.save()
    
    def test_calculations(self):
        """Test calculation methods and totals"""
        doc = self.create_test_document_with_items()
        
        # Test item calculations
        for item in doc.items:
            expected_amount = flt(item.qty) * flt(item.rate)
            self.assertEqual(flt(item.amount), expected_amount)
        
        # Test document totals
        expected_total = sum(flt(item.amount) for item in doc.items)
        self.assertEqual(flt(doc.total_amount), expected_total)
        
        # Test tax calculations (if applicable)
        if hasattr(doc, 'tax_rate'):
            expected_tax = flt(expected_total * doc.tax_rate / 100)
            self.assertEqual(flt(doc.total_tax), expected_tax)
    
    def test_status_transitions(self):
        """Test document status management"""
        doc = self.create_test_document()
        
        # Test initial status
        self.assertEqual(doc.status, "Draft")
        
        # Test submission status
        doc.submit()
        self.assertEqual(doc.status, "Submitted")
        
        # Test cancellation status  
        doc.cancel()
        self.assertEqual(doc.status, "Cancelled")
    
    def test_permissions(self):
        """Test document-level permissions"""
        doc = self.create_test_document()
        
        # Test owner permissions
        self.assertTrue(doc.has_permission("read"))
        self.assertTrue(doc.has_permission("write"))
        
        # Test role-based permissions (create test user with specific role)
        test_user = "test@example.com"
        if not frappe.db.exists("User", test_user):
            self.create_test_user(test_user, ["Sales User"])
        
        frappe.set_user(test_user)
        
        # Test read permission
        test_doc = frappe.get_doc("Your DocType", doc.name)
        self.assertTrue(test_doc.has_permission("read"))
        
        frappe.set_user("Administrator")  # Reset
    
    def test_linked_document_updates(self):
        """Test updates to linked documents"""
        doc = self.create_test_document()
        doc.submit()
        
        # Test linked document creation/update
        linked_docs = self.get_linked_documents(doc)
        
        # Verify linked documents exist
        self.assertTrue(len(linked_docs) > 0)
        
        # Test linked document values
        for linked_doc_type, linked_doc_name in linked_docs:
            linked_doc = frappe.get_doc(linked_doc_type, linked_doc_name)
            self.assertEqual(linked_doc.reference_doctype, doc.doctype)
            self.assertEqual(linked_doc.reference_name, doc.name)
    
    def test_workflow_integration(self):
        """Test workflow state transitions"""
        if not self.has_workflow():
            self.skipTest("Workflow not configured for this DocType")
        
        doc = self.create_test_document()
        
        # Test initial workflow state
        self.assertEqual(doc.workflow_state, "Draft")
        
        # Test workflow action
        doc.apply_workflow_action("Submit for Approval")
        self.assertEqual(doc.workflow_state, "Pending Approval")
    
    def test_email_notifications(self):
        """Test email notification triggers"""
        with self.patch_email_queue():
            doc = self.create_test_document()
            doc.submit()
            
            # Check if notification email was queued
            email_queue = frappe.get_all("Email Queue", 
                                       filters={"reference_doctype": doc.doctype,
                                               "reference_name": doc.name})
            self.assertTrue(len(email_queue) > 0)
    
    def test_custom_methods(self):
        """Test custom methods specific to your DocType"""
        doc = self.create_test_document()
        
        # Test custom calculation method
        result = doc.calculate_custom_total()
        self.assertIsNotNone(result)
        self.assertGreater(result, 0)
        
        # Test custom validation method
        validation_result = doc.validate_custom_rules()
        self.assertTrue(validation_result)
    
    def test_child_table_operations(self):
        """Test child table CRUD operations"""
        doc = self.create_test_document()
        
        # Test adding child records
        initial_count = len(doc.items)
        doc.append("items", {
            "item_code": self.item,
            "qty": 5,
            "rate": 100
        })
        doc.save()
        
        self.assertEqual(len(doc.items), initial_count + 1)
        
        # Test child record validation
        last_item = doc.items[-1]
        self.assertEqual(last_item.amount, 500)  # qty * rate
        
        # Test child record removal
        doc.remove(doc.items[-1])
        doc.save()
        self.assertEqual(len(doc.items), initial_count)
    
    def test_error_handling(self):
        """Test error handling and exception scenarios"""
        # Test handling of missing linked records
        with self.assertRaises(frappe.DoesNotExistError):
            doc = frappe.get_doc({
                "doctype": "Your DocType",
                "customer": "Non Existent Customer",
                "posting_date": today()
            })
            doc.insert()
        
        # Test handling of invalid data types
        with self.assertRaises(frappe.ValidationError):
            doc = self.create_test_document()
            doc.amount = "invalid_amount"  # String instead of number
            doc.save()
    
    # Helper Methods
    def create_test_document(self, **kwargs):
        """Create test document with default values"""
        default_data = {
            "doctype": "Your DocType",
            "customer": self.customer,
            "posting_date": today(),
            "company": self.company,
            "amount": 1000,
            "reference_no": frappe.generate_hash(length=8)
        }
        default_data.update(kwargs)
        
        doc = frappe.get_doc(default_data)
        doc.insert()
        return doc
    
    def create_test_document_with_items(self):
        """Create test document with child table items"""
        doc = self.create_test_document()
        
        # Add test items
        for i in range(3):
            doc.append("items", {
                "item_code": f"{self.item}_{i}",
                "qty": (i + 1) * 2,
                "rate": 100 * (i + 1)
            })
        
        doc.save()
        return doc
    
    @classmethod
    def setup_test_dependencies(cls):
        """Create test dependencies (companies, customers, items)"""
        # Create test company
        if not frappe.db.exists("Company", cls.company):
            frappe.get_doc({
                "doctype": "Company",
                "company_name": cls.company,
                "default_currency": "USD",
                "country": "United States"
            }).insert()
        
        # Create test customer
        if not frappe.db.exists("Customer", cls.customer):
            frappe.get_doc({
                "doctype": "Customer",
                "customer_name": cls.customer,
                "customer_type": "Individual"
            }).insert()
        
        # Create test items
        for i in range(3):
            item_code = f"{cls.item}_{i}"
            if not frappe.db.exists("Item", item_code):
                frappe.get_doc({
                    "doctype": "Item",
                    "item_code": item_code,
                    "item_name": f"Test Item {i}",
                    "item_group": "Test Group",
                    "stock_uom": "Nos"
                }).insert()
    
    def create_test_user(self, email, roles):
        """Create test user with specified roles"""
        if frappe.db.exists("User", email):
            return
        
        user = frappe.get_doc({
            "doctype": "User",
            "email": email,
            "first_name": "Test",
            "last_name": "User",
            "user_type": "System User"
        })
        
        for role in roles:
            user.append("roles", {"role": role})
        
        user.insert()
    
    def get_linked_documents(self, doc):
        """Get documents linked to the test document"""
        # Implementation depends on your business logic
        return []
    
    def has_workflow(self):
        """Check if workflow is configured for the DocType"""
        return frappe.db.exists("Workflow", {"document_type": "Your DocType"})
    
    def patch_email_queue(self):
        """Context manager to patch email sending for testing"""
        from unittest.mock import patch
        return patch('frappe.sendmail')

# Test Fixtures
@frappe.whitelist()
def create_test_records():
    """Create test records for manual testing"""
    records = []
    
    for i in range(5):
        doc = frappe.get_doc({
            "doctype": "Your DocType",
            "customer": "_Test Customer",
            "posting_date": add_days(today(), -i),
            "amount": 1000 * (i + 1),
            "reference_no": f"TEST-{i:03d}"
        })
        doc.insert()
        records.append(doc.name)
    
    return records

# Performance Test Helpers
def create_bulk_test_documents(count=100):
    """Create bulk documents for performance testing"""
    documents = []
    
    for i in range(count):
        doc = frappe.get_doc({
            "doctype": "Your DocType", 
            "customer": "_Test Customer",
            "posting_date": today(),
            "amount": 1000,
            "reference_no": f"BULK-{i:05d}"
        })
        doc.insert()
        documents.append(doc.name)
        
        # Commit every 50 records
        if i % 50 == 0:
            frappe.db.commit()
    
    return documents
```

---

## Integration Test Templates

### Cross-Module Integration Test Template
*Based on ERPNext's Sales Order to Sales Invoice flow*

```python
# test_integration_workflow.py
import frappe
import unittest
from frappe.utils import today, add_days

class TestIntegrationWorkflow(unittest.TestCase):
    """
    Integration tests for cross-module workflows
    Following ERPNext's integration testing patterns
    """
    
    @classmethod
    def setUpClass(cls):
        """Set up integration test environment"""
        cls.setup_integration_dependencies()
    
    def setUp(self):
        """Set up before each integration test"""
        frappe.db.rollback()
        frappe.clear_cache()
        
        # Set up test data
        self.customer = "_Test Customer"
        self.company = "_Test Company"
        self.items = ["_Test Item 1", "_Test Item 2"]
    
    def test_complete_business_flow(self):
        """Test complete business process from start to finish"""
        # Step 1: Create initial document
        source_doc = self.create_source_document()
        self.assertEqual(source_doc.status, "Draft")
        
        # Step 2: Submit source document
        source_doc.submit()
        self.assertEqual(source_doc.status, "Submitted")
        
        # Step 3: Create derived document
        target_doc = self.create_target_document_from_source(source_doc)
        self.assertIsNotNone(target_doc)
        self.assertEqual(target_doc.reference_doctype, source_doc.doctype)
        self.assertEqual(target_doc.reference_name, source_doc.name)
        
        # Step 4: Validate data mapping
        self.validate_document_mapping(source_doc, target_doc)
        
        # Step 5: Submit target document
        target_doc.submit()
        
        # Step 6: Verify back-linking
        source_doc.reload()
        self.assertEqual(source_doc.status, "Partially Processed")  # or appropriate status
        
        # Step 7: Verify system integration (GL, Stock, etc.)
        self.verify_system_integration(source_doc, target_doc)
    
    def test_cancellation_flow(self):
        """Test document cancellation and reversal flow"""
        # Create and submit documents
        source_doc = self.create_source_document()
        source_doc.submit()
        
        target_doc = self.create_target_document_from_source(source_doc)
        target_doc.submit()
        
        # Test target document cancellation
        target_doc.cancel()
        self.assertEqual(target_doc.status, "Cancelled")
        
        # Verify source document status update
        source_doc.reload()
        self.assertEqual(source_doc.status, "Submitted")
        
        # Test source document cancellation
        source_doc.cancel()
        self.assertEqual(source_doc.status, "Cancelled")
    
    def test_partial_processing(self):
        """Test partial processing scenarios"""
        source_doc = self.create_source_document_with_multiple_items()
        source_doc.submit()
        
        # Process only first item
        target_doc = self.create_partial_target_document(source_doc, process_items=[0])
        target_doc.submit()
        
        # Verify partial status
        source_doc.reload()
        self.assertEqual(source_doc.status, "Partially Processed")
        
        # Process remaining items
        target_doc_2 = self.create_partial_target_document(source_doc, process_items=[1])
        target_doc_2.submit()
        
        # Verify complete status
        source_doc.reload()
        self.assertEqual(source_doc.status, "Completed")
    
    def test_multi_company_integration(self):
        """Test integration across multiple companies"""
        company_1 = "_Test Company 1"
        company_2 = "_Test Company 2"
        
        # Create inter-company transaction
        source_doc = self.create_source_document(company=company_1)
        source_doc.submit()
        
        # Create corresponding document in second company
        target_doc = self.create_inter_company_document(source_doc, company_2)
        target_doc.submit()
        
        # Verify inter-company linking
        self.verify_inter_company_integration(source_doc, target_doc)
    
    def test_background_job_integration(self):
        """Test background job processing integration"""
        # Create document that triggers background processing
        doc = self.create_document_requiring_background_processing()
        doc.submit()
        
        # Verify background job was queued
        jobs = frappe.get_all("RQ Job", 
                            filters={"queue": "default", "status": "queued"},
                            limit=1)
        self.assertTrue(len(jobs) > 0)
        
        # Execute background job manually for testing
        self.execute_background_jobs()
        
        # Verify job completion
        doc.reload()
        self.assertEqual(doc.background_processing_status, "Completed")
    
    def test_email_integration(self):
        """Test email notification integration"""
        with self.patch_email_sending():
            # Create document that should trigger emails
            doc = self.create_document_with_email_triggers()
            doc.submit()
            
            # Verify email queue
            email_queue = frappe.get_all("Email Queue",
                                       filters={"reference_doctype": doc.doctype,
                                               "reference_name": doc.name})
            self.assertTrue(len(email_queue) > 0)
            
            # Verify email content
            email_doc = frappe.get_doc("Email Queue", email_queue[0].name)
            self.assertIn(doc.name, email_doc.message)
    
    def test_permission_integration(self):
        """Test permission inheritance across related documents"""
        # Create document with specific permissions
        source_doc = self.create_source_document()
        source_doc.submit()
        
        # Create user with limited permissions
        test_user = self.create_test_user_with_permissions()
        frappe.set_user(test_user)
        
        # Test access to source document
        self.assertTrue(frappe.has_permission("Source DocType", "read", source_doc.name))
        
        # Create target document
        target_doc = self.create_target_document_from_source(source_doc)
        
        # Verify permission inheritance
        self.assertTrue(frappe.has_permission("Target DocType", "read", target_doc.name))
        
        frappe.set_user("Administrator")  # Reset
    
    # Helper Methods
    def create_source_document(self, **kwargs):
        """Create source document for integration testing"""
        default_data = {
            "doctype": "Source DocType",
            "customer": self.customer,
            "company": kwargs.get("company", self.company),
            "posting_date": today(),
            "items": [
                {"item_code": self.items[0], "qty": 2, "rate": 100},
                {"item_code": self.items[1], "qty": 3, "rate": 150}
            ]
        }
        default_data.update(kwargs)
        
        doc = frappe.get_doc(default_data)
        doc.insert()
        return doc
    
    def create_target_document_from_source(self, source_doc):
        """Create target document from source using document mapping"""
        # Use ERPNext's document mapping pattern
        from frappe.model.mapper import get_mapped_doc
        
        target_doc = get_mapped_doc("Source DocType", source_doc.name, {
            "Source DocType": {
                "doctype": "Target DocType",
                "field_map": {
                    "name": "reference_name",
                    "customer": "customer",
                    "posting_date": "transaction_date"
                }
            },
            "Source Item": {
                "doctype": "Target Item",
                "field_map": {
                    "item_code": "item_code",
                    "qty": "qty",
                    "rate": "rate"
                }
            }
        })
        
        target_doc.insert()
        return target_doc
    
    @classmethod
    def setup_integration_dependencies(cls):
        """Set up dependencies required for integration testing"""
        # Create test companies
        companies = ["_Test Company", "_Test Company 1", "_Test Company 2"]
        for company in companies:
            if not frappe.db.exists("Company", company):
                frappe.get_doc({
                    "doctype": "Company",
                    "company_name": company,
                    "default_currency": "USD"
                }).insert()
        
        # Create test customer
        if not frappe.db.exists("Customer", "_Test Customer"):
            frappe.get_doc({
                "doctype": "Customer",
                "customer_name": "_Test Customer",
                "customer_type": "Individual"
            }).insert()
    
    def verify_system_integration(self, source_doc, target_doc):
        """Verify system-level integrations (GL, Stock, etc.)"""
        # Verify GL Entry creation
        gl_entries = frappe.get_all("GL Entry",
                                  filters={"voucher_type": target_doc.doctype,
                                          "voucher_no": target_doc.name})
        self.assertTrue(len(gl_entries) > 0)
        
        # Verify Stock Ledger Entry creation
        sle_entries = frappe.get_all("Stock Ledger Entry",
                                   filters={"voucher_type": target_doc.doctype,
                                           "voucher_no": target_doc.name})
        self.assertTrue(len(sle_entries) > 0)
```

---

## API Test Templates

### REST API Test Template
*Based on ERPNext's API testing patterns*

```python
# test_api_endpoints.py
import frappe
import unittest
import json
import requests
from frappe.test_runner import make_test_records

class TestAPIEndpoints(unittest.TestCase):
    """
    API endpoint testing following ERPNext patterns
    """
    
    @classmethod
    def setUpClass(cls):
        """Set up API testing environment"""
        cls.base_url = frappe.utils.get_url()
        cls.api_key, cls.api_secret = cls.get_api_credentials()
        cls.headers = {
            "Content-Type": "application/json",
            "Authorization": f"token {cls.api_key}:{cls.api_secret}"
        }
    
    def setUp(self):
        """Set up before each API test"""
        frappe.db.rollback()
        self.test_records = make_test_records("Your DocType")
    
    def test_get_list_endpoint(self):
        """Test GET list endpoint"""
        response = requests.get(
            f"{self.base_url}/api/resource/Your DocType",
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        
        data = response.json()
        self.assertIn("data", data)
        self.assertIsInstance(data["data"], list)
        
        # Test pagination
        response = requests.get(
            f"{self.base_url}/api/resource/Your DocType?limit=5&start=0",
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertTrue(len(data["data"]) <= 5)
    
    def test_get_single_endpoint(self):
        """Test GET single document endpoint"""
        doc_name = self.test_records[0]
        
        response = requests.get(
            f"{self.base_url}/api/resource/Your DocType/{doc_name}",
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        
        data = response.json()
        self.assertIn("data", data)
        self.assertEqual(data["data"]["name"], doc_name)
    
    def test_post_create_endpoint(self):
        """Test POST create endpoint"""
        new_doc_data = {
            "customer": "_Test Customer",
            "posting_date": "2023-01-01",
            "amount": 1000
        }
        
        response = requests.post(
            f"{self.base_url}/api/resource/Your DocType",
            headers=self.headers,
            data=json.dumps(new_doc_data)
        )
        
        self.assertEqual(response.status_code, 200)
        
        data = response.json()
        self.assertIn("data", data)
        self.assertEqual(data["data"]["customer"], "_Test Customer")
        
        # Verify document was created
        doc_name = data["data"]["name"]
        self.assertTrue(frappe.db.exists("Your DocType", doc_name))
    
    def test_put_update_endpoint(self):
        """Test PUT update endpoint"""
        doc_name = self.test_records[0]
        update_data = {
            "amount": 2000,
            "notes": "Updated via API"
        }
        
        response = requests.put(
            f"{self.base_url}/api/resource/Your DocType/{doc_name}",
            headers=self.headers,
            data=json.dumps(update_data)
        )
        
        self.assertEqual(response.status_code, 200)
        
        # Verify update
        doc = frappe.get_doc("Your DocType", doc_name)
        self.assertEqual(doc.amount, 2000)
        self.assertEqual(doc.notes, "Updated via API")
    
    def test_delete_endpoint(self):
        """Test DELETE endpoint"""
        doc_name = self.test_records[-1]  # Use last record
        
        response = requests.delete(
            f"{self.base_url}/api/resource/Your DocType/{doc_name}",
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 202)
        
        # Verify deletion
        self.assertFalse(frappe.db.exists("Your DocType", doc_name))
    
    def test_api_authentication(self):
        """Test API authentication scenarios"""
        # Test without authentication
        response = requests.get(f"{self.base_url}/api/resource/Your DocType")
        self.assertEqual(response.status_code, 401)
        
        # Test with invalid credentials
        invalid_headers = {
            "Authorization": "token invalid:credentials"
        }
        response = requests.get(
            f"{self.base_url}/api/resource/Your DocType",
            headers=invalid_headers
        )
        self.assertEqual(response.status_code, 401)
    
    def test_api_permissions(self):
        """Test API permission enforcement"""
        # Create limited user
        limited_user = self.create_limited_api_user()
        limited_headers = {
            "Authorization": f"token {limited_user['api_key']}:{limited_user['api_secret']}"
        }
        
        # Test read access
        response = requests.get(
            f"{self.base_url}/api/resource/Your DocType",
            headers=limited_headers
        )
        self.assertEqual(response.status_code, 200)
        
        # Test write access (should fail for limited user)
        response = requests.post(
            f"{self.base_url}/api/resource/Your DocType",
            headers=limited_headers,
            data=json.dumps({"customer": "_Test Customer"})
        )
        self.assertEqual(response.status_code, 403)
    
    def test_api_filtering(self):
        """Test API filtering and search"""
        # Test field filtering
        response = requests.get(
            f"{self.base_url}/api/resource/Your DocType?filters=[['customer','=','_Test Customer']]",
            headers=self.headers
        )
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        for record in data["data"]:
            self.assertEqual(record["customer"], "_Test Customer")
    
    def test_custom_api_methods(self):
        """Test custom API methods"""
        doc_name = self.test_records[0]
        
        # Test custom method call
        response = requests.post(
            f"{self.base_url}/api/method/your_app.api.custom_method",
            headers=self.headers,
            data=json.dumps({"doc_name": doc_name, "action": "process"})
        )
        
        self.assertEqual(response.status_code, 200)
        
        data = response.json()
        self.assertIn("message", data)
    
    @classmethod
    def get_api_credentials(cls):
        """Get API credentials for testing"""
        # Create or get test user with API access
        user_email = "api.test@example.com"
        
        if not frappe.db.exists("User", user_email):
            user = frappe.get_doc({
                "doctype": "User",
                "email": user_email,
                "first_name": "API",
                "last_name": "Test",
                "user_type": "System User"
            })
            user.append("roles", {"role": "System Manager"})
            user.insert()
        else:
            user = frappe.get_doc("User", user_email)
        
        # Generate API keys
        if not user.api_key:
            api_key = frappe.generate_hash(length=15)
            api_secret = frappe.generate_hash(length=15)
            
            user.api_key = api_key
            user.api_secret = api_secret
            user.save()
        
        return user.api_key, user.api_secret
    
    def create_limited_api_user(self):
        """Create user with limited API permissions"""
        user_email = "limited.api.test@example.com"
        
        if frappe.db.exists("User", user_email):
            frappe.delete_doc("User", user_email)
        
        user = frappe.get_doc({
            "doctype": "User",
            "email": user_email,
            "first_name": "Limited",
            "last_name": "API User",
            "user_type": "System User"
        })
        user.append("roles", {"role": "Employee"})  # Limited role
        user.insert()
        
        # Generate API keys
        api_key = frappe.generate_hash(length=15)
        api_secret = frappe.generate_hash(length=15)
        
        user.api_key = api_key
        user.api_secret = api_secret
        user.save()
        
        return {
            "api_key": api_key,
            "api_secret": api_secret
        }
```

This comprehensive test template system provides production-ready testing patterns based on ERPNext's proven testing framework. The templates cover all aspects of testing from unit tests to API integration tests.

**Key Features:**

1. **Complete Test Coverage** - Unit, integration, and API tests
2. **ERPNext Patterns** - Based on actual ERPNext test implementations
3. **Test Data Management** - Proper setup and teardown procedures
4. **Permission Testing** - Role-based access control testing
5. **Performance Patterns** - Bulk operations and performance testing
6. **Production-Ready** - Error handling and edge case testing

**Usage:**
1. Copy the appropriate test template for your testing needs
2. Customize test data and business logic validation
3. Add DocType-specific test cases
4. Run tests regularly as part of your CI/CD pipeline
5. Use for test-driven development practices

These templates ensure your applications maintain the same testing standards as ERPNext's enterprise-grade codebase.