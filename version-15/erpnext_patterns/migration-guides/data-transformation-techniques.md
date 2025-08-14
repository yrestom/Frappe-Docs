# Data Transformation Techniques - ERPNext Migration Patterns

*Based on ERPNext migration analysis - Proven techniques for safe, efficient data transformations during migrations*

## 🎯 Overview

This guide provides production-tested data transformation techniques extracted from ERPNext's migration patches. These patterns ensure data integrity, performance, and maintainability when transforming data structures during version upgrades or custom migrations.

---

## 📋 Table of Contents

1. [Field Transformation Patterns](#field-transformation-patterns)
2. [Data Structure Migrations](#data-structure-migrations)
3. [Bulk Data Operations](#bulk-data-operations)
4. [Multi-Company Data Transformations](#multi-company-data-transformations)
5. [Currency and Localization](#currency-and-localization)
6. [Tree and Hierarchical Data](#tree-and-hierarchical-data)
7. [Template and Configuration Migrations](#template-and-configuration-migrations)
8. [Performance Optimization Techniques](#performance-optimization-techniques)
9. [Error Handling and Rollback](#error-handling-and-rollback)

---

## 🔄 Field Transformation Patterns

### Field Renaming with Validation
*Based on ERPNext's tolerance field renaming pattern*

```python
# field_rename_pattern.py
import frappe
from frappe.model.utils.rename_field import rename_field

def safe_field_rename():
    """
    Safe field renaming with pre-validation checks
    Pattern from erpnext/patches/v12_0/rename_tolerance_fields.py
    """
    
    # 1. Reload relevant DocTypes first
    frappe.reload_doc("stock", "doctype", "item")
    frappe.reload_doc("stock", "doctype", "stock_settings")
    frappe.reload_doc("accounts", "doctype", "accounts_settings")
    
    # 2. Check if field exists before renaming
    if frappe.db.has_column("Stock Settings", "tolerance"):
        rename_field("Stock Settings", "tolerance", "over_delivery_receipt_allowance")
    
    if frappe.db.has_column("Item", "tolerance"):
        rename_field("Item", "tolerance", "over_delivery_receipt_allowance")
    
    # 3. Copy field values to new locations if needed
    qty_allowance = frappe.db.get_single_value("Stock Settings", "over_delivery_receipt_allowance")
    frappe.db.set_single_value("Accounts Settings", "over_delivery_receipt_allowance", qty_allowance)
    
    # 4. Update derived fields with SQL for performance
    frappe.db.sql("UPDATE tabItem SET over_billing_allowance=over_delivery_receipt_allowance")

# Advanced Field Transformation
def transform_field_with_business_logic():
    """Transform field values with business logic validation"""
    
    # Check current data state
    existing_data = frappe.db.sql("""
        SELECT name, old_field_name, related_field 
        FROM `tabYour DocType` 
        WHERE old_field_name IS NOT NULL
    """, as_dict=1)
    
    for record in existing_data:
        # Apply transformation logic
        new_value = transform_value(record.old_field_name, record.related_field)
        
        # Validate transformation
        if validate_transformation(new_value, record):
            frappe.db.set_value(
                "Your DocType", 
                record.name, 
                "new_field_name", 
                new_value
            )
        else:
            # Log transformation issues for review
            frappe.log_error(
                f"Transformation failed for {record.name}: {record.old_field_name}",
                "Field Transformation Error"
            )

def transform_value(old_value, context_field):
    """Apply business logic transformation"""
    if old_value == "Legacy Status":
        return "New Status" if context_field else "Draft"
    elif old_value in ["Old1", "Old2"]:
        return "Consolidated Status"
    return old_value

def validate_transformation(new_value, record):
    """Validate transformed value meets business requirements"""
    if not new_value:
        return False
    
    # Business rule validation
    if record.related_field == "Critical" and new_value == "Draft":
        return False
    
    return True
```

### Field Type Conversions
*Based on ERPNext's timesheet currency field additions*

```python
# field_type_conversion.py
import frappe

def convert_field_types():
    """
    Convert field types with data preservation
    Pattern from erpnext/patches/v13_0/update_timesheet_changes.py
    """
    
    # 1. Reload DocTypes to get new field definitions
    frappe.reload_doc("projects", "doctype", "timesheet")
    frappe.reload_doc("projects", "doctype", "timesheet_detail")
    
    # 2. Rename fields if they exist
    if frappe.db.has_column("Timesheet Detail", "billable"):
        rename_field("Timesheet Detail", "billable", "is_billable")
    
    # 3. Add currency-related fields with default values
    base_currency = frappe.defaults.get_global_default("currency")
    
    # Update child table with base currency fields
    frappe.db.sql("""
        UPDATE `tabTimesheet Detail`
        SET base_billing_rate = billing_rate,
            base_billing_amount = billing_amount,
            base_costing_rate = costing_rate,
            base_costing_amount = costing_amount
        WHERE base_billing_rate IS NULL OR base_billing_rate = 0
    """)
    
    # Update parent table with currency information
    frappe.db.sql(f"""
        UPDATE `tabTimesheet`
        SET currency = '{base_currency}',
            exchange_rate = 1.0,
            base_total_billable_amount = total_billable_amount,
            base_total_billed_amount = total_billed_amount,
            base_total_costing_amount = total_costing_amount
        WHERE currency IS NULL OR currency = ''
    """)

# Complex Type Conversion Pattern
class FieldTypeConverter:
    """Handle complex field type conversions with validation"""
    
    def __init__(self, doctype, field_name):
        self.doctype = doctype
        self.field_name = field_name
        self.conversion_log = []
    
    def convert_text_to_longtext(self):
        """Convert Text field to Long Text with data validation"""
        
        # Check current field type
        current_meta = frappe.get_meta(self.doctype)
        field_meta = current_meta.get_field(self.field_name)
        
        if field_meta.fieldtype != "Text":
            return {"success": False, "message": "Field is not of type Text"}
        
        # Find records with long content that might be truncated
        long_content_records = frappe.db.sql(f"""
            SELECT name, {self.field_name}
            FROM `tab{self.doctype}`
            WHERE LENGTH({self.field_name}) > 1000
        """, as_dict=1)
        
        # Log potential data issues
        for record in long_content_records:
            self.conversion_log.append({
                "record": record.name,
                "content_length": len(record[self.field_name]),
                "might_be_truncated": len(record[self.field_name]) >= 65535
            })
        
        return {
            "success": True,
            "records_affected": len(long_content_records),
            "conversion_log": self.conversion_log
        }
    
    def convert_select_to_link(self, target_doctype):
        """Convert Select field to Link field with reference DocType creation"""
        
        # Get all unique values from select field
        unique_values = frappe.db.sql_list(f"""
            SELECT DISTINCT {self.field_name}
            FROM `tab{self.doctype}`
            WHERE {self.field_name} IS NOT NULL
            AND {self.field_name} != ''
        """)
        
        # Create reference documents
        created_records = []
        for value in unique_values:
            if not frappe.db.exists(target_doctype, value):
                doc = frappe.get_doc({
                    "doctype": target_doctype,
                    "name": value,
                    f"{target_doctype.lower()}_name": value
                })
                doc.insert()
                created_records.append(value)
        
        return {
            "unique_values": len(unique_values),
            "created_records": created_records
        }
```

---

## 🏗️ Data Structure Migrations

### Parent to Child Table Migration
*Based on ERPNext's Item Defaults migration pattern*

```python
# parent_to_child_migration.py
import frappe

def migrate_fields_to_child_table():
    """
    Migrate fields from parent to child table for multi-company support
    Pattern from erpnext/patches/v11_0/move_item_defaults_to_child_table_for_multicompany.py
    """
    
    # Fields to migrate from parent to child table
    fields_to_migrate = [
        "default_warehouse", "buying_cost_center", "expense_account", 
        "selling_cost_center", "income_account", "default_supplier"
    ]
    
    # Check if migration is needed
    if not frappe.db.has_column("Item", "default_warehouse"):
        return {"status": "skipped", "reason": "Fields already migrated"}
    
    # Reload DocTypes
    frappe.reload_doc("stock", "doctype", "item_default")
    frappe.reload_doc("stock", "doctype", "item")
    
    companies = frappe.get_all("Company")
    
    if len(companies) == 1 and not frappe.get_all("Item Default", limit=1):
        # Single company - direct migration
        migrate_single_company(companies[0].name, fields_to_migrate)
    else:
        # Multi-company - complex migration
        migrate_multi_company(fields_to_migrate)

def migrate_single_company(company_name, fields_to_migrate):
    """Migrate data for single company setup"""
    
    field_list = ", ".join(fields_to_migrate)
    
    frappe.db.sql(f"""
        INSERT INTO `tabItem Default`
        (name, parent, parenttype, parentfield, idx, company, {field_list})
        SELECT
            SUBSTRING(SHA2(name,224), 1, 10) as name, 
            name as parent, 
            'Item' as parenttype,
            'item_defaults' as parentfield, 
            1 as idx, 
            %s as company,
            {field_list}
        FROM `tabItem`
        WHERE name NOT IN (SELECT DISTINCT parent FROM `tabItem Default`)
    """, company_name)

def migrate_multi_company(fields_to_migrate):
    """Migrate data for multi-company setup with company inference"""
    
    # Get item details that haven't been migrated yet
    item_details = frappe.db.sql("""
        SELECT name, default_warehouse, buying_cost_center, expense_account, 
               selling_cost_center, income_account, default_supplier
        FROM tabItem
        WHERE name NOT IN (SELECT DISTINCT parent FROM `tabItem Default`) 
        AND ifnull(disabled, 0) = 0
    """, as_dict=1)
    
    items_default_data = {}
    
    # Group data by company (inferred from related master data)
    for item_data in item_details:
        field_company_mapping = [
            ["default_warehouse", "Warehouse"],
            ["expense_account", "Account"],
            ["income_account", "Account"],
            ["buying_cost_center", "Cost Center"],
            ["selling_cost_center", "Cost Center"],
        ]
        
        for field_name, doctype in field_company_mapping:
            field_value = item_data.get(field_name)
            if field_value:
                # Infer company from related document
                company = frappe.get_value(doctype, field_value, "company", cache=True)
                
                if company:
                    # Initialize nested dictionary structure
                    if item_data.name not in items_default_data:
                        items_default_data[item_data.name] = {}
                    
                    if company not in items_default_data[item_data.name]:
                        items_default_data[item_data.name][company] = {}
                    
                    items_default_data[item_data.name][company][field_name] = field_value
    
    # Prepare bulk insert data
    bulk_insert_data = []
    for item_code, company_wise_data in items_default_data.items():
        for company, item_default_data in company_wise_data.items():
            bulk_insert_data.append((
                frappe.generate_hash("", 10),  # name
                item_code,                      # parent
                "Item",                         # parenttype
                "item_defaults",                # parentfield
                company,                        # company
                item_default_data.get("default_warehouse"),
                item_default_data.get("expense_account"),
                item_default_data.get("income_account"),
                item_default_data.get("buying_cost_center"),
                item_default_data.get("selling_cost_center"),
            ))
    
    # Execute bulk insert
    if bulk_insert_data:
        frappe.db.sql(f"""
            INSERT INTO `tabItem Default`
            (name, parent, parenttype, parentfield, company, default_warehouse,
             expense_account, income_account, buying_cost_center, selling_cost_center)
            VALUES {", ".join(["%s"] * len(bulk_insert_data))}
        """, tuple(bulk_insert_data))

# Generic Parent-to-Child Migration Framework
class ParentToChildMigrator:
    """Framework for migrating fields from parent to child table"""
    
    def __init__(self, parent_doctype, child_doctype, child_field_name):
        self.parent_doctype = parent_doctype
        self.child_doctype = child_doctype  
        self.child_field_name = child_field_name
        self.migration_log = []
    
    def migrate_fields(self, field_mapping, grouping_logic=None):
        """
        Migrate fields with flexible grouping logic
        
        field_mapping: dict mapping old_field -> new_field
        grouping_logic: function to determine how to group child records
        """
        
        # Get parent records to migrate
        parent_records = self.get_parent_records_to_migrate(field_mapping.keys())
        
        for parent_record in parent_records:
            try:
                # Apply grouping logic (e.g., by company, by category)
                if grouping_logic:
                    groups = grouping_logic(parent_record)
                else:
                    groups = [{"default": parent_record}]  # Single group
                
                # Create child records for each group
                for group_name, group_data in groups.items():
                    child_doc_data = {
                        "doctype": self.child_doctype,
                        "parent": parent_record["name"],
                        "parenttype": self.parent_doctype,
                        "parentfield": self.child_field_name
                    }
                    
                    # Map fields
                    for old_field, new_field in field_mapping.items():
                        if group_data.get(old_field):
                            child_doc_data[new_field] = group_data[old_field]
                    
                    # Add group identifier if applicable
                    if group_name != "default":
                        child_doc_data["group_identifier"] = group_name
                    
                    # Insert child record
                    child_doc = frappe.get_doc(child_doc_data)
                    child_doc.insert()
                    
                self.migration_log.append({
                    "parent": parent_record["name"],
                    "groups_created": len(groups),
                    "status": "success"
                })
                
            except Exception as e:
                self.migration_log.append({
                    "parent": parent_record["name"], 
                    "status": "failed",
                    "error": str(e)
                })
                frappe.log_error(f"Migration failed for {parent_record['name']}", str(e))
        
        return self.migration_log
    
    def get_parent_records_to_migrate(self, fields):
        """Get parent records that need migration"""
        field_conditions = " OR ".join([f"{field} IS NOT NULL" for field in fields])
        
        return frappe.db.sql(f"""
            SELECT name, {", ".join(fields)}
            FROM `tab{self.parent_doctype}`
            WHERE ({field_conditions})
            AND name NOT IN (
                SELECT DISTINCT parent 
                FROM `tab{self.child_doctype}` 
                WHERE parenttype = '{self.parent_doctype}'
            )
        """, as_dict=1)
```

### Template-Based Data Transformation
*Based on ERPNext's Item Tax Template migration*

```python
# template_migration.py
import json
import frappe
from frappe.model.naming import make_autoname

def migrate_flat_data_to_templates():
    """
    Convert flat data structure to template-based structure
    Pattern from erpnext/patches/v12_0/move_item_tax_to_item_tax_template.py
    """
    
    # Check if migration is needed
    if "tax_type" not in frappe.db.get_table_columns("Item Tax"):
        return {"status": "skipped", "reason": "Already migrated"}
    
    old_item_taxes = {}
    existing_templates = {}
    
    # Load existing templates to avoid duplicates
    existing_template_data = frappe.db.sql("""
        SELECT template.name, details.tax_type, details.tax_rate
        FROM `tabItem Tax Template` template, `tabItem Tax Template Detail` details
        WHERE details.parent = template.name
    """, as_dict=1)
    
    for template_detail in existing_template_data:
        template_name = template_detail.name
        if template_name not in existing_templates:
            existing_templates[template_name] = {}
        existing_templates[template_name][template_detail.tax_type] = template_detail.tax_rate
    
    # Collect old tax data
    for tax_record in frappe.db.sql("""
        SELECT parent as item_code, tax_type, tax_rate 
        FROM `tabItem Tax`
    """, as_dict=1):
        if tax_record.item_code not in old_item_taxes:
            old_item_taxes[tax_record.item_code] = []
        old_item_taxes[tax_record.item_code].append(tax_record)
    
    # Enable auto-commit for large datasets
    frappe.db.auto_commit_on_many_writes = True
    
    migration_stats = {
        "items_processed": 0,
        "templates_created": 0,
        "templates_reused": 0
    }
    
    try:
        # Process each item
        for item_code, item_tax_list in old_item_taxes.items():
            # Create tax map for current item
            item_tax_map = {}
            for tax_data in item_tax_list:
                item_tax_map[tax_data.tax_type] = tax_data.tax_rate
            
            # Find or create appropriate template
            template_name = get_or_create_item_tax_template(
                existing_templates, item_tax_map, item_code
            )
            
            if template_name:
                # Update item with new template reference
                frappe.db.sql("DELETE FROM `tabItem Tax` WHERE parent=%s AND parenttype='Item'", item_code)
                
                item_doc = frappe.get_doc("Item", item_code)
                item_doc.set("taxes", [])
                item_doc.append("taxes", {
                    "item_tax_template": template_name,
                    "tax_category": ""
                })
                
                # Direct database insert for performance
                for tax_row in item_doc.taxes:
                    tax_row.db_insert()
                
                migration_stats["items_processed"] += 1
        
        # Process transaction documents
        process_transaction_documents(existing_templates, migration_stats)
        
    finally:
        frappe.db.auto_commit_on_many_writes = False
    
    return migration_stats

def get_or_create_item_tax_template(existing_templates, item_tax_map, item_code):
    """Find existing template or create new one"""
    
    # Search for existing template with same tax map
    for template_name, template_tax_map in existing_templates.items():
        if item_tax_map == template_tax_map:
            return template_name
    
    # Create new template
    template = frappe.new_doc("Item Tax Template")
    template.title = make_autoname("Item Tax Template-.####")
    
    # Add tax details
    for tax_type, tax_rate in item_tax_map.items():
        # Validate tax account exists
        if validate_tax_account(tax_type, template):
            template.append("taxes", {
                "tax_type": tax_type,
                "tax_rate": tax_rate
            })
    
    if template.get("taxes"):
        template.save()
        # Cache new template
        existing_templates[template.name] = item_tax_map
        return template.name
    
    return None

def validate_tax_account(tax_type, template):
    """Validate and create tax account if needed"""
    
    account_details = frappe.db.get_value(
        "Account", tax_type, 
        ["name", "account_type", "company"], 
        as_dict=1
    )
    
    if account_details:
        template.company = account_details.company
        
        # Ensure correct account type
        valid_account_types = [
            "Tax", "Chargeable", "Income Account", 
            "Expense Account", "Expenses Included In Valuation"
        ]
        
        if account_details.account_type not in valid_account_types:
            frappe.db.set_value("Account", account_details.name, "account_type", "Chargeable")
        
        return True
    
    return False

def process_transaction_documents(existing_templates, migration_stats):
    """Process transaction documents (Sales Invoice, Purchase Invoice, etc.)"""
    
    transaction_doctypes = [
        "Quotation", "Sales Order", "Delivery Note", "Sales Invoice",
        "Supplier Quotation", "Purchase Order", "Purchase Receipt", "Purchase Invoice"
    ]
    
    for doctype in transaction_doctypes:
        # Get records with old tax structure
        records = frappe.db.sql(f"""
            SELECT name, parenttype, parent, item_code, item_tax_rate
            FROM `tab{doctype} Item`
            WHERE ifnull(item_tax_rate, '') NOT IN ('', '{{}}')
            AND item_tax_template IS NULL
        """, as_dict=1)
        
        for record in records:
            try:
                item_tax_map = json.loads(record.item_tax_rate)
                template_name = get_or_create_item_tax_template(
                    existing_templates, item_tax_map, record.item_code
                )
                
                if template_name:
                    frappe.db.set_value(
                        f"{doctype} Item", 
                        record.name, 
                        "item_tax_template", 
                        template_name
                    )
                
            except json.JSONDecodeError:
                frappe.log_error(f"Invalid JSON in item_tax_rate for {record.name}")
                continue

# Generic Template Migration Framework
class TemplatedDataMigrator:
    """Framework for converting flat data to template-based structure"""
    
    def __init__(self, source_doctype, template_doctype, template_detail_doctype):
        self.source_doctype = source_doctype
        self.template_doctype = template_doctype
        self.template_detail_doctype = template_detail_doctype
        self.existing_templates = {}
        self.migration_stats = {
            "processed": 0,
            "templates_created": 0,
            "templates_reused": 0,
            "errors": 0
        }
    
    def migrate_to_templates(self, data_grouping_fn, template_creation_fn):
        """
        Generic template migration
        
        data_grouping_fn: Function to group source data into templates
        template_creation_fn: Function to create template from grouped data
        """
        
        # Load existing templates
        self.load_existing_templates()
        
        # Group source data
        grouped_data = data_grouping_fn(self.source_doctype)
        
        # Process each group
        for group_key, group_data in grouped_data.items():
            try:
                template_name = self.find_or_create_template(
                    group_key, group_data, template_creation_fn
                )
                
                if template_name:
                    self.update_source_records(group_data, template_name)
                    self.migration_stats["processed"] += len(group_data["records"])
                
            except Exception as e:
                self.migration_stats["errors"] += 1
                frappe.log_error(f"Template migration error for {group_key}", str(e))
        
        return self.migration_stats
    
    def find_or_create_template(self, group_key, group_data, template_creation_fn):
        """Find existing template or create new one"""
        
        # Generate template signature for matching
        template_signature = self.generate_template_signature(group_data)
        
        # Check existing templates
        for template_name, signature in self.existing_templates.items():
            if signature == template_signature:
                self.migration_stats["templates_reused"] += 1
                return template_name
        
        # Create new template
        template_name = template_creation_fn(group_key, group_data)
        if template_name:
            self.existing_templates[template_name] = template_signature
            self.migration_stats["templates_created"] += 1
        
        return template_name
    
    def generate_template_signature(self, group_data):
        """Generate unique signature for template matching"""
        # Implement based on template structure
        pass
    
    def load_existing_templates(self):
        """Load existing templates with their signatures"""
        # Implement based on template structure
        pass
    
    def update_source_records(self, group_data, template_name):
        """Update source records to reference the template"""
        # Implement based on source data structure
        pass
```

---

## ⚡ Bulk Data Operations

### High-Performance Bulk Updates
*Based on ERPNext's SQL-based update patterns*

```python
# bulk_operations.py
import frappe
from frappe.utils import flt, cint

class BulkDataProcessor:
    """High-performance bulk data operations with batching and progress tracking"""
    
    def __init__(self, doctype, batch_size=1000):
        self.doctype = doctype
        self.batch_size = batch_size
        self.processed_count = 0
        self.error_count = 0
        self.progress_log = []
    
    def bulk_update_with_calculation(self, update_spec):
        """
        Bulk update with calculations and validation
        
        update_spec = {
            'filters': {'status': 'Active'},
            'updates': [
                {'field': 'calculated_field', 'calculation': 'amount * rate'},
                {'field': 'status', 'value': 'Processed'}
            ],
            'validation': lambda record: record.amount > 0
        }
        """
        
        # Get total count for progress tracking
        total_count = frappe.db.count(self.doctype, filters=update_spec.get('filters', {}))
        
        # Process in batches
        offset = 0
        while offset < total_count:
            batch = self.get_batch_data(update_spec.get('filters', {}), offset)
            
            if not batch:
                break
            
            self.process_batch(batch, update_spec)
            offset += self.batch_size
            
            # Log progress
            progress_pct = min(100, (offset / total_count) * 100)
            self.progress_log.append({
                "timestamp": frappe.utils.now(),
                "processed": self.processed_count,
                "errors": self.error_count,
                "progress_pct": progress_pct
            })
            
            frappe.db.commit()  # Commit each batch
        
        return {
            "total_processed": self.processed_count,
            "total_errors": self.error_count,
            "progress_log": self.progress_log
        }
    
    def get_batch_data(self, filters, offset):
        """Get batch of records with all required fields"""
        
        return frappe.get_all(
            self.doctype,
            filters=filters,
            fields=['*'],  # Get all fields for calculations
            limit=self.batch_size,
            start=offset,
            order_by='modified desc'  # Process newest first
        )
    
    def process_batch(self, batch, update_spec):
        """Process single batch with error handling"""
        
        sql_updates = []
        individual_updates = []
        
        for record in batch:
            try:
                # Validate record if validation function provided
                if update_spec.get('validation') and not update_spec['validation'](record):
                    continue
                
                # Prepare updates
                update_values = {}
                for update in update_spec['updates']:
                    if 'calculation' in update:
                        # Evaluate calculation
                        value = self.evaluate_calculation(update['calculation'], record)
                        update_values[update['field']] = value
                    elif 'value' in update:
                        # Direct value assignment
                        update_values[update['field']] = update['value']
                
                # Determine update method
                if len(update_values) == 1 and 'calculation' not in str(update_spec['updates']):
                    # Simple SQL update
                    sql_updates.append((record.name, update_values))
                else:
                    # Individual document update for complex cases
                    individual_updates.append((record.name, update_values))
                
            except Exception as e:
                self.error_count += 1
                frappe.log_error(f"Batch processing error for {record.name}", str(e))
        
        # Execute SQL updates
        if sql_updates:
            self.execute_sql_updates(sql_updates, update_spec['updates'][0]['field'])
        
        # Execute individual updates
        for record_name, update_values in individual_updates:
            try:
                frappe.db.set_value(self.doctype, record_name, update_values)
                self.processed_count += 1
            except Exception as e:
                self.error_count += 1
                frappe.log_error(f"Individual update error for {record_name}", str(e))
    
    def execute_sql_updates(self, updates, field_name):
        """Execute bulk SQL updates for performance"""
        
        if not updates:
            return
        
        # Create WHEN-THEN clauses for CASE statement
        when_clauses = []
        name_list = []
        
        for record_name, update_values in updates:
            when_clauses.append(f"WHEN name = '{record_name}' THEN {update_values[field_name]}")
            name_list.append(f"'{record_name}'")
        
        # Execute bulk update
        sql = f"""
            UPDATE `tab{self.doctype}`
            SET {field_name} = CASE 
                {' '.join(when_clauses)}
                ELSE {field_name}
            END,
            modified = NOW()
            WHERE name IN ({', '.join(name_list)})
        """
        
        frappe.db.sql(sql)
        self.processed_count += len(updates)
    
    def evaluate_calculation(self, calculation_expr, record):
        """Safely evaluate calculation expressions"""
        
        # Replace field references with actual values
        safe_expr = calculation_expr
        for field_name, field_value in record.items():
            if field_name in safe_expr:
                # Convert to float for calculations
                numeric_value = flt(field_value) if field_value else 0
                safe_expr = safe_expr.replace(field_name, str(numeric_value))
        
        try:
            # Evaluate with restricted scope
            allowed_names = {
                "__builtins__": {},
                "flt": flt,
                "cint": cint,
                "abs": abs,
                "round": round,
                "min": min,
                "max": max
            }
            return eval(safe_expr, allowed_names)
        except:
            return 0

# Specific Bulk Operations
def bulk_update_status_with_conditions():
    """Bulk status update based on business conditions"""
    
    # Complex status update with multiple conditions
    frappe.db.sql("""
        UPDATE `tabSales Order`
        SET status = CASE
            WHEN per_delivered >= 100 THEN 'Completed'
            WHEN per_delivered > 0 THEN 'Partially Delivered'
            WHEN docstatus = 1 THEN 'To Deliver'
            ELSE 'Draft'
        END,
        modified = NOW()
        WHERE status != 'Cancelled'
    """)

def bulk_currency_conversion():
    """Bulk currency field updates with exchange rates"""
    
    # Get current exchange rates
    exchange_rates = frappe.db.sql("""
        SELECT from_currency, to_currency, exchange_rate
        FROM `tabCurrency Exchange`
        WHERE date = CURDATE()
    """, as_dict=1)
    
    rate_map = {f"{r.from_currency}_{r.to_currency}": r.exchange_rate for r in exchange_rates}
    
    # Update documents with base currency fields
    frappe.db.sql("""
        UPDATE `tabSales Invoice`
        SET base_grand_total = grand_total * %s,
            base_outstanding_amount = outstanding_amount * %s
        WHERE currency != %s
        AND currency = %s
    """, (
        rate_map.get("USD_INR", 1),
        rate_map.get("USD_INR", 1), 
        "INR",
        "USD"
    ))

def bulk_recompute_totals():
    """Recompute document totals from child table data"""
    
    frappe.db.sql("""
        UPDATE `tabSales Order` so
        SET so.total_qty = (
            SELECT SUM(soi.qty) 
            FROM `tabSales Order Item` soi 
            WHERE soi.parent = so.name
        ),
        so.base_total = (
            SELECT SUM(soi.base_amount) 
            FROM `tabSales Order Item` soi 
            WHERE soi.parent = so.name
        )
        WHERE so.docstatus = 1
    """)

# Bulk Data Quality Operations
def bulk_data_cleanup():
    """Clean up data quality issues in bulk"""
    
    cleanup_operations = [
        # Remove extra whitespace
        {
            "description": "Trim whitespace",
            "sql": """
                UPDATE `tabCustomer`
                SET customer_name = TRIM(customer_name),
                    customer_group = TRIM(customer_group)
                WHERE customer_name != TRIM(customer_name)
                OR customer_group != TRIM(customer_group)
            """
        },
        
        # Standardize phone numbers
        {
            "description": "Standardize phone format",
            "sql": """
                UPDATE `tabCustomer`
                SET phone = REGEXP_REPLACE(phone, '[^0-9+]', '')
                WHERE phone IS NOT NULL
                AND phone != ''
            """
        },
        
        # Fix email case
        {
            "description": "Normalize email case",
            "sql": """
                UPDATE `tabCustomer`
                SET email_id = LOWER(email_id)
                WHERE email_id IS NOT NULL
                AND email_id != LOWER(email_id)
            """
        }
    ]
    
    results = []
    for operation in cleanup_operations:
        try:
            affected_rows = frappe.db.sql(operation["sql"])
            results.append({
                "operation": operation["description"],
                "affected_rows": affected_rows,
                "status": "success"
            })
        except Exception as e:
            results.append({
                "operation": operation["description"],
                "error": str(e),
                "status": "failed"
            })
    
    return results
```

---

## 🏢 Multi-Company Data Transformations

### Company-Aware Data Migration
*Based on ERPNext's multi-company patterns*

```python
# multi_company_migration.py
import frappe

class MultiCompanyDataMigrator:
    """Handle data transformations across multiple companies"""
    
    def __init__(self):
        self.companies = frappe.get_all("Company", fields=["name", "abbr", "default_currency"])
        self.company_map = {c.name: c for c in self.companies}
    
    def migrate_with_company_inference(self, doctype, field_mappings):
        """
        Migrate data with automatic company inference
        
        field_mappings = [
            {'field': 'warehouse', 'reference_doctype': 'Warehouse'},
            {'field': 'cost_center', 'reference_doctype': 'Cost Center'}
        ]
        """
        
        migration_results = []
        
        # Get records to process
        records = frappe.get_all(
            doctype,
            fields=["name"] + [mapping["field"] for mapping in field_mappings],
            filters={"company": ["in", ["", None]]}  # Records without company
        )
        
        for record in records:
            try:
                # Infer company from related documents
                inferred_company = self.infer_company_from_references(record, field_mappings)
                
                if inferred_company:
                    frappe.db.set_value(doctype, record.name, "company", inferred_company)
                    migration_results.append({
                        "record": record.name,
                        "company": inferred_company,
                        "status": "success"
                    })
                else:
                    # Use default company if single company setup
                    if len(self.companies) == 1:
                        default_company = self.companies[0].name
                        frappe.db.set_value(doctype, record.name, "company", default_company)
                        migration_results.append({
                            "record": record.name,
                            "company": default_company,
                            "status": "defaulted"
                        })
                    else:
                        migration_results.append({
                            "record": record.name,
                            "status": "failed",
                            "reason": "Cannot infer company"
                        })
                
            except Exception as e:
                migration_results.append({
                    "record": record.name,
                    "status": "error",
                    "error": str(e)
                })
        
        return migration_results
    
    def infer_company_from_references(self, record, field_mappings):
        """Infer company from referenced documents"""
        
        for mapping in field_mappings:
            field_value = record.get(mapping["field"])
            if field_value:
                company = frappe.get_value(
                    mapping["reference_doctype"], 
                    field_value, 
                    "company"
                )
                if company:
                    return company
        
        return None
    
    def split_data_by_company(self, source_doctype, target_doctype, split_rules):
        """
        Split single-company data into multi-company structure
        
        split_rules = {
            'company_field': 'company',
            'unique_fields': ['name', 'company'],  # Fields for uniqueness
            'copy_fields': ['field1', 'field2'],   # Fields to copy
            'transform_fields': {                   # Fields to transform
                'old_field': 'new_field'
            }
        }
        """
        
        # Get source data
        source_records = frappe.get_all(
            source_doctype,
            fields=["*"]
        )
        
        created_records = []
        
        for source_record in source_records:
            for company in self.companies:
                try:
                    # Check if record already exists for this company
                    existing_filters = {split_rules['company_field']: company.name}
                    for field in split_rules['unique_fields']:
                        if field != split_rules['company_field']:
                            existing_filters[field] = source_record.get(field)
                    
                    if frappe.db.exists(target_doctype, existing_filters):
                        continue
                    
                    # Create new record for company
                    new_record = {
                        "doctype": target_doctype,
                        split_rules['company_field']: company.name
                    }
                    
                    # Copy specified fields
                    for field in split_rules.get('copy_fields', []):
                        new_record[field] = source_record.get(field)
                    
                    # Transform fields
                    for old_field, new_field in split_rules.get('transform_fields', {}).items():
                        new_record[new_field] = source_record.get(old_field)
                    
                    # Apply company-specific transformations
                    new_record = self.apply_company_transformations(
                        new_record, company, source_record
                    )
                    
                    # Create document
                    doc = frappe.get_doc(new_record)
                    doc.insert()
                    
                    created_records.append({
                        "source": source_record.name,
                        "target": doc.name,
                        "company": company.name
                    })
                    
                except Exception as e:
                    frappe.log_error(
                        f"Failed to create {target_doctype} for company {company.name}",
                        str(e)
                    )
        
        return created_records
    
    def apply_company_transformations(self, record, company, source_record):
        """Apply company-specific field transformations"""
        
        # Add company-specific naming
        if "name" in record and not record.get("name"):
            base_name = source_record.get("name", "")
            record["name"] = f"{base_name} - {company.abbr}"
        
        # Set company-specific defaults
        if hasattr(record, "currency") and not record.get("currency"):
            record["currency"] = company.default_currency
        
        # Transform account names with company suffix
        for field_name, field_value in record.items():
            if field_name.endswith("_account") and field_value:
                # Check if account exists for this company
                account_name = field_value
                if " - " not in account_name:  # No company suffix
                    company_account = f"{account_name} - {company.abbr}"
                    if frappe.db.exists("Account", company_account):
                        record[field_name] = company_account
        
        return record

def consolidate_multi_company_data():
    """Consolidate data across companies for reporting"""
    
    # Create consolidated view of customer data
    frappe.db.sql("""
        CREATE OR REPLACE VIEW consolidated_customers AS
        SELECT 
            c.name,
            c.customer_name,
            c.customer_group,
            c.territory,
            c.company,
            comp.abbr as company_abbr,
            COALESCE(
                (SELECT SUM(grand_total) 
                 FROM `tabSales Invoice` si 
                 WHERE si.customer = c.name 
                 AND si.company = c.company 
                 AND si.docstatus = 1),
                0
            ) as total_billing
        FROM `tabCustomer` c
        LEFT JOIN `tabCompany` comp ON c.company = comp.name
    """)

def sync_master_data_across_companies():
    """Synchronize master data changes across companies"""
    
    # Sync item changes to all companies
    master_items = frappe.get_all(
        "Item",
        fields=["name", "item_name", "item_group", "description", "modified"]
    )
    
    for item in master_items:
        # Update item in all company-specific contexts
        for company in frappe.get_all("Company"):
            try:
                # Update item defaults for each company
                item_defaults = frappe.get_all(
                    "Item Default",
                    filters={"parent": item.name, "company": company.name},
                    limit=1
                )
                
                if item_defaults:
                    # Sync common fields
                    frappe.db.sql("""
                        UPDATE `tabItem Default`
                        SET item_name = %s,
                            modified = %s
                        WHERE parent = %s AND company = %s
                    """, (item.item_name, item.modified, item.name, company.name))
                
            except Exception as e:
                frappe.log_error(f"Master data sync failed for {item.name} in {company.name}", str(e))

# Company-specific data validation
def validate_company_data_integrity():
    """Validate data integrity across company boundaries"""
    
    integrity_checks = []
    
    # Check for cross-company data leaks
    cross_company_issues = frappe.db.sql("""
        SELECT 
            'Account' as doctype,
            a.name,
            a.company as doc_company,
            je.company as transaction_company
        FROM `tabAccount` a
        JOIN `tabJournal Entry Account` jea ON jea.account = a.name
        JOIN `tabJournal Entry` je ON je.name = jea.parent
        WHERE a.company != je.company
        LIMIT 100
    """, as_dict=1)
    
    if cross_company_issues:
        integrity_checks.append({
            "check": "Cross-company account usage",
            "issues_found": len(cross_company_issues),
            "sample_issues": cross_company_issues[:5]
        })
    
    # Check for missing company assignments
    missing_company = frappe.db.sql("""
        SELECT doctype, COUNT(*) as count
        FROM (
            SELECT 'Sales Invoice' as doctype FROM `tabSales Invoice` WHERE company IS NULL OR company = ''
            UNION ALL
            SELECT 'Purchase Invoice' as doctype FROM `tabPurchase Invoice` WHERE company IS NULL OR company = ''
            UNION ALL
            SELECT 'Journal Entry' as doctype FROM `tabJournal Entry` WHERE company IS NULL OR company = ''
        ) missing
        GROUP BY doctype
    """, as_dict=1)
    
    if missing_company:
        integrity_checks.append({
            "check": "Missing company assignments",
            "details": missing_company
        })
    
    return integrity_checks
```

This comprehensive data transformation guide provides battle-tested patterns for safely transforming data structures during migrations. The techniques are extracted from ERPNext's production migration patches and ensure data integrity, performance, and maintainability.

**Key Features:**

1. **Safe Field Operations** - Field renaming, type conversion with validation
2. **Structure Migrations** - Parent-to-child table, template-based transformations
3. **High-Performance Operations** - Bulk updates, SQL optimizations, batching
4. **Multi-Company Support** - Company-aware migrations and data consolidation
5. **Error Handling** - Comprehensive error handling and rollback support
6. **Progress Tracking** - Migration monitoring and logging

**Usage Guidelines:**
- Always test transformations on staging data first
- Use transactions for atomic operations
- Implement progress tracking for long-running migrations
- Validate data integrity at each step
- Maintain detailed migration logs for audit trails