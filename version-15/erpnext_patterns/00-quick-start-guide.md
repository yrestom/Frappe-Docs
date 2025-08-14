# Quick Start Guide - ERPNext Development

*Fast-track development setup using ERPNext's proven patterns - Get productive in 30 minutes*

## 🚀 Overview

This guide gets you up and running with ERPNext development using production-tested patterns extracted from the ERPNext codebase. Follow these steps to create your first enterprise-grade Frappe application.

---

## ⚡ 30-Minute Quick Start

### Step 1: Environment Setup (5 minutes)
```bash
# Install Frappe bench if not already installed
pip install frappe-bench

# Create new site (replace with your domain)
bench new-site mysite.local
bench use mysite.local

# Install ERPNext for reference patterns
bench get-app erpnext
bench install-app erpnext

# Create your custom app
bench new-app my_custom_app
bench install-app my_custom_app
```

### Step 2: App Structure Setup (10 minutes)

**1. Configure hooks.py** (Based on ERPNext patterns)
```python
# my_custom_app/hooks.py
from . import __version__ as app_version

app_name = "my_custom_app"
app_title = "My Custom App"
app_publisher = "Your Organization"
app_description = "Custom business application built on Frappe"
app_email = "admin@yourorg.com"
app_license = "MIT"

# Document Events (Based on ERPNext patterns)
doc_events = {
    "User": {
        "after_insert": "my_custom_app.utils.create_user_profile"
    },
    "Company": {
        "on_update": "my_custom_app.setup.company.update_custom_settings"
    }
}

# Scheduled Tasks (Following ERPNext frequency patterns)
scheduler_events = {
    "daily": [
        "my_custom_app.tasks.daily.send_daily_reports"
    ],
    "weekly": [
        "my_custom_app.tasks.weekly.cleanup_old_data"
    ]
}

# Website context (If web views needed)
website_context = {
    "favicon": "/assets/my_custom_app/images/favicon.png",
    "splash_image": "/assets/my_custom_app/images/splash.png"
}
```

**2. Create module structure**
```bash
mkdir -p my_custom_app/my_module
echo "My Module" > my_custom_app/modules.txt
```

### Step 3: Create Your First DocType (10 minutes)

**1. Transaction DocType (Following Sales Invoice patterns)**
```bash
# Create DocType via UI or programmatically
bench execute "
import frappe
from my_custom_app.templates.doctype_creator import create_transaction_doctype
create_transaction_doctype('Project Task', 'My Module')
"
```

**2. Python Controller** (Based on TransactionBase patterns)
```python
# my_custom_app/my_module/doctype/project_task/project_task.py
import frappe
from frappe import _
from frappe.model.document import Document
from frappe.utils import flt, getdate, add_days, today

class ProjectTask(Document):
    def validate(self):
        self.validate_dates()
        self.validate_task_details()
        self.calculate_totals()
        self.set_status()
    
    def validate_dates(self):
        """Validate date logic following ERPNext patterns"""
        if self.start_date and self.end_date:
            if getdate(self.start_date) > getdate(self.end_date):
                frappe.throw(_("Start Date cannot be after End Date"))
    
    def validate_task_details(self):
        """Business validation following ERPNext validation patterns"""
        if not self.project:
            frappe.throw(_("Project is mandatory"))
        
        if flt(self.expected_hours) <= 0:
            frappe.throw(_("Expected Hours must be greater than 0"))
    
    def calculate_totals(self):
        """Calculate totals following ERPNext calculation patterns"""
        self.progress_percentage = flt(self.actual_hours / self.expected_hours * 100) if self.expected_hours else 0
    
    def set_status(self):
        """Status management following ERPNext status patterns"""
        if self.progress_percentage == 100:
            self.status = "Completed"
        elif self.progress_percentage > 0:
            self.status = "In Progress"
        else:
            self.status = "Open"
```

**3. JavaScript Controller** (Based on ERPNext client patterns)
```javascript
// my_custom_app/my_module/doctype/project_task/project_task.js
frappe.ui.form.on('Project Task', {
    refresh: function(frm) {
        // Add custom buttons following ERPNext patterns
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__('Create Timesheet'), function() {
                frm.trigger('make_timesheet');
            });
        }
        
        // Set indicators following ERPNext patterns
        frm.set_indicator_based_on_status();
    },
    
    project: function(frm) {
        // Auto-fetch project details following ERPNext patterns
        if (frm.doc.project) {
            frappe.call({
                method: 'frappe.client.get_value',
                args: {
                    doctype: 'Project',
                    name: frm.doc.project,
                    fieldname: ['customer', 'company', 'project_type']
                },
                callback: function(r) {
                    if (r.message) {
                        frm.set_value('customer', r.message.customer);
                        frm.set_value('company', r.message.company);
                    }
                }
            });
        }
    },
    
    make_timesheet: function(frm) {
        // Document mapping following ERPNext patterns
        frappe.model.open_mapped_doc({
            method: 'my_custom_app.my_module.doctype.project_task.project_task.make_timesheet',
            frm: frm
        });
    }
});

// Extend form with custom functionality
frappe.ui.form.ProjectTask = frappe.ui.form.Controller.extend({
    set_indicator_based_on_status: function() {
        let color_map = {
            'Open': 'red',
            'In Progress': 'orange', 
            'Completed': 'green'
        };
        this.frm.page.set_indicator(__(this.frm.doc.status), color_map[this.frm.doc.status]);
    }
});
```

### Step 4: Test Your Implementation (5 minutes)

**1. Create test data**
```python
# test_project_task.py
import frappe
import unittest
from frappe.utils import today, add_days

class TestProjectTask(unittest.TestCase):
    def test_task_validation(self):
        """Test task validation following ERPNext test patterns"""
        task = frappe.get_doc({
            "doctype": "Project Task",
            "task_name": "Test Task",
            "project": "Test Project",
            "expected_hours": 8,
            "start_date": today(),
            "end_date": add_days(today(), 7)
        })
        task.insert()
        
        # Test status calculation
        task.actual_hours = 4
        task.save()
        self.assertEqual(task.progress_percentage, 50)
        self.assertEqual(task.status, "In Progress")
```

**2. Run tests**
```bash
bench run-tests --app my_custom_app --doctype "Project Task"
```

---

## 🎯 Essential Patterns Cheat Sheet

### DocType Design Patterns
```python
# Master DocType Pattern (Customer/Item style)
- Use "field:master_name" for naming
- Include status, disabled fields
- Add contact integration
- Implement dashboard data

# Transaction DocType Pattern (Sales Order/Invoice style)  
- Use "naming_series:" for naming
- Include company, posting_date fields
- Add items table with calculations
- Implement submission workflow

# Child Table Pattern (Invoice Item style)
- Set istable: 1
- Include parent reference fields
- Add calculation methods
- Validate against parent data
```

### Controller Patterns
```python
# Validation Flow (Following ERPNext patterns)
def validate(self):
    self.validate_basic_info()      # Basic field validation
    self.validate_business_rules()  # Business logic validation
    self.calculate_values()         # Calculations
    self.set_missing_values()       # Auto-set defaults
    self.validate_final_checks()    # Final validations

# Status Management (Following ERPNext patterns)  
def set_status(self):
    if self.docstatus == 2:
        self.status = "Cancelled"
    elif self.docstatus == 1:
        self.status = "Submitted"
    else:
        self.status = self.get_draft_status()
```

### JavaScript Patterns
```javascript
// Form Enhancement Pattern
frappe.ui.form.on('DocType', {
    refresh: function(frm) {
        frm.add_custom_buttons();    // Add action buttons
        frm.set_form_indicators();   // Status indicators
        frm.toggle_field_display();  // Conditional display
    },
    
    setup: function(frm) {
        frm.set_queries();          // Configure link filters
        frm.setup_buttons();        // Initialize buttons
    }
});

// Child Table Pattern
frappe.ui.form.on('Child DocType', {
    qty: function(frm, cdt, cdn) {
        calculate_amount(frm, cdt, cdn);    // Recalculate on change
        frm.trigger('calculate_totals');     // Update parent totals
    }
});
```

---

## 🛠️ Development Workflow

### Daily Development Process
1. **Start with ERPNext pattern analysis** - Find similar functionality in ERPNext
2. **Copy proven patterns** - Use templates from this documentation
3. **Customize for your needs** - Modify business logic while keeping structure
4. **Test thoroughly** - Use ERPNext testing patterns
5. **Deploy with confidence** - Following ERPNext deployment practices

### Key Resources
- 📖 **Core Documentation**: Start with `01-erpnext-architecture-analysis.md`
- 🧩 **Templates**: Use templates from `templates/` directory
- ✅ **Quality**: Follow `checklists/development-checklist.md`
- 🔧 **Troubleshooting**: Check `troubleshooting-index.md`

---

## 🎉 What's Next?

### Immediate Next Steps (Choose based on your needs)
- **Building Complex Transactions?** → Read `03-business-logic-implementation.md`
- **Adding Client-side Features?** → Study `04-client-side-development-patterns.md`  
- **Need Reporting?** → Explore `09-reporting-and-analytics.md`
- **Planning Deployment?** → Review `11-deployment-and-operations.md`

### Progressive Learning Path
1. **Week 1**: Master DocType and Controller patterns
2. **Week 2**: Implement client-side enhancements and workflows  
3. **Week 3**: Add reporting and analytics features
4. **Week 4**: Implement security, testing, and deployment

### Get Help
- **Documentation Issues**: Refer to specific analysis files
- **Pattern Questions**: Check cross-reference matrix in master index
- **Implementation Problems**: Use troubleshooting index

---

## ⚠️ Important Notes

**Production Readiness**: This quick start gets you functional quickly, but review the full documentation before production deployment.

**Security**: Implement proper permission controls before exposing to users.

**Testing**: Always test thoroughly, especially business logic and calculations.

**Performance**: Monitor and optimize based on usage patterns.

---

*This quick start guide condenses years of ERPNext development wisdom into actionable steps. You're now equipped with production-tested patterns used by thousands of ERPNext installations worldwide.*

**Ready to dive deeper? Explore the comprehensive analysis files for advanced patterns and enterprise-grade implementations.**