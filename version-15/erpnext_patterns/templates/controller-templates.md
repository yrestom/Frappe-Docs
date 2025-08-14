# Controller Templates - Production-Ready Patterns

*Based on ERPNext controller analysis - Production-tested patterns extracted from AccountsController, SellingController, and BuyingController*

## Table of Contents

1. [Base Controller Template](#base-controller-template)
2. [Transaction Controller Template](#transaction-controller-template)
3. [Financial Controller Template](#financial-controller-template)
4. [Stock Controller Template](#stock-controller-template)
5. [Workflow Controller Template](#workflow-controller-template)
6. [API Controller Template](#api-controller-template)

---

## Base Controller Template

*Based on frappe.model.document.Document and ERPNext core patterns*

```python
# Copyright (c) 2025, Your Organization and contributors
# For license information, please see license.txt

import frappe
from frappe import _
from frappe.model.document import Document
from frappe.utils import (
    flt, cint, cstr, get_link_to_form, today, nowdate, nowtime,
    add_days, add_months, getdate, validate_email_address
)

class BaseController(Document):
    """
    Base controller with common functionality for all DocTypes
    Based on ERPNext core patterns and best practices
    """
    
    def __init__(self, *args, **kwargs):
        super(BaseController, self).__init__(*args, **kwargs)
        self.status_field = "status"  # Override in subclasses if different
        self.title_field = None       # Override in subclasses if applicable
        
    def validate(self):
        """Master validation method - override in subclasses"""
        self.set_missing_values()
        self.validate_dates()
        self.validate_mandatory_fields()
        self.validate_duplicate_entries()
        self.set_title()
        
    def before_save(self):
        """Pre-save processing"""
        self.set_defaults()
        self.clean_data()
        
    def after_save(self):
        """Post-save processing"""
        self.update_modified_by_info()
        
    def on_update(self):
        """Post-update processing"""
        self.clear_cache()
        self.update_linked_documents()
        
    def validate_dates(self):
        """Validate date fields following ERPNext patterns"""
        date_fields = self.get_date_fields()
        
        for field in date_fields:
            if self.get(field):
                try:
                    getdate(self.get(field))
                except:
                    frappe.throw(_("Invalid date format in field {0}")
                               .format(self.meta.get_label(field)))
        
        # Validate date ranges if applicable
        self.validate_date_ranges()
        
    def validate_date_ranges(self):
        """Validate date ranges - override in subclasses"""
        # Example: from_date should be before to_date
        if hasattr(self, 'from_date') and hasattr(self, 'to_date'):
            if self.from_date and self.to_date:
                if getdate(self.from_date) > getdate(self.to_date):
                    frappe.throw(_("From Date cannot be after To Date"))
                    
    def get_date_fields(self):
        """Get all date/datetime fields in the DocType"""
        return [df.fieldname for df in self.meta.fields 
                if df.fieldtype in ["Date", "Datetime"]]
        
    def validate_mandatory_fields(self):
        """Enhanced mandatory field validation"""
        # Get conditional mandatory fields
        conditional_mandatory = self.get_conditional_mandatory_fields()
        
        for field_name, condition in conditional_mandatory.items():
            if self.evaluate_condition(condition) and not self.get(field_name):
                frappe.throw(_("{0} is mandatory").format(
                    self.meta.get_label(field_name)
                ))
                
    def get_conditional_mandatory_fields(self):
        """Get fields that are conditionally mandatory - override in subclasses"""
        return {}
        
    def evaluate_condition(self, condition):
        """Evaluate conditional logic"""
        if callable(condition):
            return condition(self)
        elif isinstance(condition, str):
            return eval(condition, {"self": self, "frappe": frappe})
        return bool(condition)
        
    def validate_duplicate_entries(self):
        """Validate duplicate entries based on unique constraints"""
        unique_constraints = self.get_unique_constraints()
        
        for constraint in unique_constraints:
            self.check_duplicate_constraint(constraint)
            
    def get_unique_constraints(self):
        """Get unique constraints - override in subclasses"""
        return []
        
    def check_duplicate_constraint(self, fields):
        """Check for duplicate entries based on field combination"""
        if isinstance(fields, str):
            fields = [fields]
            
        filters = {field: self.get(field) for field in fields if self.get(field)}
        filters["name"] = ["!=", self.name]
        
        if frappe.db.exists(self.doctype, filters):
            field_labels = [self.meta.get_label(field) for field in fields]
            frappe.throw(_("Duplicate entry for {0}").format(", ".join(field_labels)))
            
    def set_missing_values(self):
        """Set missing values with intelligent defaults"""
        self.set_company_defaults()
        self.set_user_defaults()
        self.set_calculated_fields()
        
    def set_company_defaults(self):
        """Set company-specific defaults"""
        if hasattr(self, 'company') and not self.company:
            self.company = frappe.defaults.get_user_default("Company")
            
        if self.company:
            company_defaults = self.get_company_defaults()
            for field, value in company_defaults.items():
                if not self.get(field):
                    self.set(field, value)
                    
    def get_company_defaults(self):
        """Get company-specific default values - override in subclasses"""
        if not hasattr(self, 'company') or not self.company:
            return {}
            
        return frappe.get_cached_value("Company", self.company, [
            "default_currency", "country", "default_letter_head"
        ], as_dict=True) or {}
        
    def set_user_defaults(self):
        """Set user-specific defaults"""
        user_defaults = frappe.defaults.get_user_defaults()
        
        for field, value in user_defaults.items():
            if hasattr(self, field) and not self.get(field):
                self.set(field, value)
                
    def set_calculated_fields(self):
        """Set calculated/computed fields - override in subclasses"""
        pass
        
    def set_title(self):
        """Set document title for display"""
        if self.title_field and hasattr(self, self.title_field):
            if not self.get(self.title_field):
                title = self.generate_title()
                if title:
                    self.set(self.title_field, title)
                    
    def generate_title(self):
        """Generate document title - override in subclasses"""
        return None
        
    def set_defaults(self):
        """Set default values before save"""
        # Set default status
        if hasattr(self, self.status_field) and not self.get(self.status_field):
            self.set(self.status_field, "Active")
            
        # Set creation date fields
        if hasattr(self, 'date') and not self.date:
            self.date = today()
            
    def clean_data(self):
        """Clean and normalize data"""
        # Clean string fields
        string_fields = [df.fieldname for df in self.meta.fields 
                        if df.fieldtype in ["Data", "Text"]]
        
        for field in string_fields:
            if self.get(field):
                self.set(field, cstr(self.get(field)).strip())
                
        # Validate email fields
        email_fields = [df.fieldname for df in self.meta.fields 
                       if df.options == "Email"]
        
        for field in email_fields:
            if self.get(field):
                validate_email_address(self.get(field), throw=True)
                
    def update_modified_by_info(self):
        """Update modification tracking"""
        if hasattr(self, 'modified_by_info'):
            self.modified_by_info = frappe.session.user
            
    def clear_cache(self):
        """Clear relevant caches after update"""
        # Clear document cache
        frappe.cache().delete_key(f"doc_{self.doctype}_{self.name}")
        
        # Clear related caches - override in subclasses
        self.clear_related_caches()
        
    def clear_related_caches(self):
        """Clear caches related to this document - override in subclasses"""
        pass
        
    def update_linked_documents(self):
        """Update linked documents after changes"""
        linked_docs = self.get_linked_documents()
        
        for doctype, doc_names in linked_docs.items():
            for doc_name in doc_names:
                self.update_linked_document(doctype, doc_name)
                
    def get_linked_documents(self):
        """Get documents linked to this record - override in subclasses"""
        return {}
        
    def update_linked_document(self, doctype, doc_name):
        """Update a specific linked document"""
        try:
            doc = frappe.get_doc(doctype, doc_name)
            if hasattr(doc, 'update_totals'):
                doc.update_totals()
                doc.save(ignore_permissions=True)
        except Exception as e:
            frappe.log_error(f"Failed to update {doctype} {doc_name}", str(e))
            
    def get_permission_query_conditions(user):
        """Permission query conditions - override in subclasses"""
        if not user:
            user = frappe.session.user
            
        if "System Manager" in frappe.get_roles(user):
            return None
            
        # Default: user can access their own company's records
        companies = frappe.get_list("Company", 
                                  filters={"name": ["in", frappe.get_user().get_companies()]},
                                  pluck="name")
        
        if companies:
            return f"""company in ({', '.join(f"'{c}'" for c in companies)})"""
        
        return "1=0"  # No access if no companies
        
    def has_permission(doc, ptype, user):
        """Document-level permissions - override in subclasses"""
        if not user:
            user = frappe.session.user
            
        if "System Manager" in frappe.get_roles(user):
            return True
            
        # Default: check company access
        if hasattr(doc, 'company') and doc.company:
            user_companies = frappe.get_user().get_companies()
            return doc.company in user_companies
            
        return True
        
    def get_dashboard_data(self):
        """Get dashboard data for this document"""
        return {
            "fieldname": self.get_dashboard_fieldname(),
            "transactions": self.get_dashboard_transactions(),
            "reports": self.get_dashboard_reports()
        }
        
    def get_dashboard_fieldname(self):
        """Get field name for dashboard linking - override in subclasses"""
        return "name"
        
    def get_dashboard_transactions(self):
        """Get transaction groups for dashboard - override in subclasses"""
        return []
        
    def get_dashboard_reports(self):
        """Get reports for dashboard - override in subclasses"""
        return []
        
    def send_notification(self, event, recipients=None):
        """Send notifications for document events"""
        if not recipients:
            recipients = self.get_notification_recipients(event)
            
        if recipients:
            subject = self.get_notification_subject(event)
            message = self.get_notification_message(event)
            
            frappe.sendmail(
                recipients=recipients,
                subject=subject,
                message=message,
                reference_doctype=self.doctype,
                reference_name=self.name,
                attachments=self.get_notification_attachments(event)
            )
            
    def get_notification_recipients(self, event):
        """Get notification recipients for event - override in subclasses"""
        return []
        
    def get_notification_subject(self, event):
        """Get notification subject - override in subclasses"""
        return _("{0} {1} - {2}").format(self.doctype, self.name, event.title())
        
    def get_notification_message(self, event):
        """Get notification message - override in subclasses"""
        return _("Document {0} {1} has been {2}")
               .format(self.doctype, self.name, event)
               
    def get_notification_attachments(self, event):
        """Get notification attachments - override in subclasses"""
        return []
```

---

## Transaction Controller Template

*Based on TransactionBase, AccountsController, and SellingController patterns*

```python
# Copyright (c) 2025, Your Organization and contributors
# For license information, please see license.txt

import frappe
from frappe import _
from frappe.utils import flt, cint, getdate, add_days
from your_module.controllers.base_controller import BaseController

class TransactionController(BaseController):
    """
    Transaction controller for submittable documents
    Based on ERPNext TransactionBase patterns
    """
    
    def __init__(self, *args, **kwargs):
        super(TransactionController, self).__init__(*args, **kwargs)
        self.item_table = "items"  # Override if different
        
    def validate(self):
        """Transaction-specific validation"""
        super(TransactionController, self).validate()
        self.validate_posting_date()
        self.validate_items()
        self.validate_currency()
        self.calculate_totals()
        self.round_floats_in_doc(2)
        
    def before_submit(self):
        """Pre-submission validations and processing"""
        self.validate_for_submit()
        self.update_status_on_submit()
        self.set_posting_date_and_time()
        
    def on_submit(self):
        """Post-submission processing"""
        self.create_auto_entries()
        self.update_linked_documents_on_submit()
        self.send_notification("submitted")
        
    def before_cancel(self):
        """Pre-cancellation validations"""
        self.validate_for_cancel()
        
    def on_cancel(self):
        """Post-cancellation processing"""
        self.cancel_auto_entries()
        self.update_linked_documents_on_cancel()
        self.update_status_on_cancel()
        self.send_notification("cancelled")
        
    def validate_posting_date(self):
        """Validate posting date against fiscal year and company settings"""
        if not self.posting_date:
            frappe.throw(_("Posting Date is mandatory"))
            
        # Check if posting date is in a closed period
        self.check_if_posting_date_is_closed()
        
        # Validate fiscal year
        self.validate_fiscal_year()
        
    def check_if_posting_date_is_closed(self):
        """Check if posting date falls in a closed accounting period"""
        closed_date = frappe.db.get_value("Company", self.company, "period_closing_date")
        
        if closed_date and getdate(self.posting_date) <= getdate(closed_date):
            frappe.throw(_("Posting Date cannot be before period closing date {0}")
                       .format(closed_date))
                       
    def validate_fiscal_year(self):
        """Validate posting date against fiscal year"""
        from erpnext.accounts.utils import get_fiscal_year
        
        try:
            fiscal_year = get_fiscal_year(self.posting_date, company=self.company)
            self.fiscal_year = fiscal_year[0]
        except Exception:
            frappe.throw(_("Posting Date {0} is not within any active fiscal year")
                       .format(self.posting_date))
                       
    def validate_items(self):
        """Validate items table"""
        if not self.get(self.item_table):
            frappe.throw(_("Items table cannot be empty"))
            
        for idx, item in enumerate(self.get(self.item_table)):
            self.validate_item_details(item, idx)
            
    def validate_item_details(self, item, idx):
        """Validate individual item details"""
        if not item.item_code:
            frappe.throw(_("Item Code is mandatory in row {0}").format(idx + 1))
            
        if flt(item.qty) <= 0:
            frappe.throw(_("Quantity must be greater than 0 in row {0}")
                       .format(idx + 1))
                       
        if flt(item.rate) < 0:
            frappe.throw(_("Rate cannot be negative in row {0}")
                       .format(idx + 1))
                       
        # Calculate amount
        item.amount = flt(item.qty) * flt(item.rate)
        
        # Validate item-specific rules
        self.validate_item_specific_rules(item, idx)
        
    def validate_item_specific_rules(self, item, idx):
        """Validate item-specific business rules - override in subclasses"""
        pass
        
    def validate_currency(self):
        """Validate currency and exchange rates"""
        if not self.currency:
            if self.company:
                self.currency = frappe.get_cached_value("Company", 
                                                      self.company, 
                                                      "default_currency")
            else:
                self.currency = frappe.defaults.get_global_default("currency")
                
        # Set exchange rate
        self.set_exchange_rate()
        
    def set_exchange_rate(self):
        """Set exchange rate for multi-currency transactions"""
        company_currency = frappe.get_cached_value("Company", 
                                                 self.company, 
                                                 "default_currency")
        
        if self.currency == company_currency:
            self.conversion_rate = 1.0
        else:
            self.conversion_rate = self.get_exchange_rate()
            
    def get_exchange_rate(self):
        """Get exchange rate for currency conversion"""
        from erpnext.setup.utils import get_exchange_rate
        
        return get_exchange_rate(
            self.currency,
            frappe.get_cached_value("Company", self.company, "default_currency"),
            self.posting_date
        ) or 1.0
        
    def calculate_totals(self):
        """Calculate document totals"""
        self.calculate_item_totals()
        self.calculate_taxes()
        self.calculate_grand_total()
        
    def calculate_item_totals(self):
        """Calculate item-level totals"""
        self.total_qty = 0
        self.base_total = 0
        self.net_total = 0
        
        for item in self.get(self.item_table, []):
            # Calculate item amount
            item.amount = flt(item.qty) * flt(item.rate)
            item.base_amount = flt(item.amount) * flt(self.conversion_rate)
            
            # Add to totals
            self.total_qty += flt(item.qty)
            self.base_total += flt(item.base_amount)
            self.net_total += flt(item.amount)
            
    def calculate_taxes(self):
        """Calculate taxes and charges - override in subclasses for specific tax logic"""
        self.total_taxes_and_charges = 0
        self.base_total_taxes_and_charges = 0
        
        # Simplified tax calculation - extend based on requirements
        if hasattr(self, 'tax_rate') and self.tax_rate:
            self.total_taxes_and_charges = flt(self.net_total * self.tax_rate / 100)
            self.base_total_taxes_and_charges = flt(self.total_taxes_and_charges * self.conversion_rate)
            
    def calculate_grand_total(self):
        """Calculate grand totals"""
        self.grand_total = flt(self.net_total + self.total_taxes_and_charges)
        self.base_grand_total = flt(self.base_total + self.base_total_taxes_and_charges)
        
        # Round grand totals
        self.rounded_total = round(self.grand_total)
        self.base_rounded_total = round(self.base_grand_total)
        
        # Calculate rounding adjustment
        self.rounding_adjustment = flt(self.rounded_total - self.grand_total)
        self.base_rounding_adjustment = flt(self.base_rounded_total - self.base_grand_total)
        
    def round_floats_in_doc(self, precision=2):
        """Round float values to specified precision"""
        float_fields = [df.fieldname for df in self.meta.fields 
                       if df.fieldtype in ["Currency", "Float"]]
        
        for field in float_fields:
            if self.get(field):
                self.set(field, flt(self.get(field), precision))
                
        # Round child table values
        for table_field in [df.fieldname for df in self.meta.fields if df.fieldtype == "Table"]:
            for item in self.get(table_field, []):
                for df in frappe.get_meta(df.options).fields:
                    if df.fieldtype in ["Currency", "Float"] and item.get(df.fieldname):
                        item.set(df.fieldname, flt(item.get(df.fieldname), precision))
                        
    def set_posting_date_and_time(self):
        """Set posting date and time if not already set"""
        if not self.posting_time:
            from frappe.utils import nowtime
            self.posting_time = nowtime()
            
        if not self.posting_date:
            self.posting_date = today()
            
    def validate_for_submit(self):
        """Validations before submission"""
        if flt(self.grand_total) == 0:
            frappe.throw(_("Grand Total cannot be zero"))
            
        # Check for mandatory approvals
        self.check_approval_status()
        
    def check_approval_status(self):
        """Check if document has required approvals"""
        if self.requires_approval() and not self.is_approved():
            frappe.throw(_("Document requires approval before submission"))
            
    def requires_approval(self):
        """Check if document requires approval - override in subclasses"""
        return False
        
    def is_approved(self):
        """Check if document is approved - override in subclasses"""
        return True
        
    def validate_for_cancel(self):
        """Validations before cancellation"""
        # Check if document can be cancelled
        if not self.can_cancel():
            frappe.throw(_("Document cannot be cancelled"))
            
    def can_cancel(self):
        """Check if document can be cancelled - override in subclasses"""
        return True
        
    def update_status_on_submit(self):
        """Update status when document is submitted"""
        self.status = "Submitted"
        
    def update_status_on_cancel(self):
        """Update status when document is cancelled"""
        self.status = "Cancelled"
        
    def create_auto_entries(self):
        """Create automatic entries on submission - override in subclasses"""
        pass
        
    def cancel_auto_entries(self):
        """Cancel automatic entries on cancellation - override in subclasses"""
        pass
        
    def update_linked_documents_on_submit(self):
        """Update linked documents on submission - override in subclasses"""
        pass
        
    def update_linked_documents_on_cancel(self):
        """Update linked documents on cancellation - override in subclasses"""
        pass

# Utility Functions for Transaction Controllers
def get_item_details(item_code, company=None):
    """Get item details for transactions"""
    if not item_code:
        return {}
        
    item = frappe.get_cached_doc("Item", item_code)
    
    details = {
        "item_name": item.item_name,
        "description": item.description,
        "stock_uom": item.stock_uom,
        "standard_rate": item.standard_rate or 0,
        "item_group": item.item_group,
        "brand": item.brand
    }
    
    # Get company-specific details
    if company:
        # Add company-specific item details if needed
        pass
        
    return details
    
def validate_conversion_rate(currency, conversion_rate, posting_date, company):
    """Validate conversion rate for multi-currency transactions"""
    if not conversion_rate or conversion_rate <= 0:
        frappe.throw(_("Conversion rate must be greater than 0"))
        
    company_currency = frappe.get_cached_value("Company", company, "default_currency")
    
    if currency != company_currency:
        # Get market rate for comparison
        from erpnext.setup.utils import get_exchange_rate
        market_rate = get_exchange_rate(currency, company_currency, posting_date)
        
        if market_rate and abs(conversion_rate - market_rate) / market_rate > 0.05:  # 5% tolerance
            frappe.msgprint(_("Conversion rate {0} seems to be different from market rate {1}")
                          .format(conversion_rate, market_rate))
```

This controller template system provides production-ready patterns based on ERPNext's proven transaction handling architecture. The templates include comprehensive validation, multi-currency support, tax calculations, and proper event handling that follows ERPNext's established patterns.

**Key Features:**

1. **Multi-layer Validation**: Progressive validation from base to specific business rules
2. **Financial Controls**: Fiscal year validation, period closing checks
3. **Multi-currency Support**: Exchange rate handling and currency conversions
4. **Event-driven Architecture**: Proper submission/cancellation lifecycle
5. **Extensible Design**: Easy to override for specific business requirements
6. **Error Handling**: Comprehensive error messages and validation feedback

**Usage:**

1. Inherit from the appropriate controller template
2. Override specific methods for custom business logic
3. Implement tax calculations, GL entries, or stock movements as needed
4. Add approval workflows and notification systems
5. Test thoroughly with various scenarios