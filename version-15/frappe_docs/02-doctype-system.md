# DocType System - Comprehensive Guide

> **Complete reference for Frappe's metadata-driven document management system**

## Table of Contents

- [What are DocTypes?](#what-are-doctypes)
- [DocType Architecture](#doctype-architecture)
- [Field Types & Properties](#field-types--properties)
- [DocType Creation & Management](#doctype-creation--management)
- [Document Lifecycle](#document-lifecycle)
- [Relationships & Links](#relationships--links)
- [Naming & Auto-naming](#naming--auto-naming)
- [Permissions & Security](#permissions--security)
- [Advanced Features](#advanced-features)
- [Best Practices](#best-practices)

## What are DocTypes?

**DocTypes** are the cornerstone of Frappe's metadata-driven architecture. They define the structure, behavior, and relationships of data entities in your application.

### Core Concept

```python
# From frappe/core/doctype/doctype/doctype.py:88
class DocType(Document):
    """
    DocType defines the metadata for all documents in the system.
    It's literally a "type of document" - like User, Item, Sales Order, etc.
    """
```

**Think of DocTypes as:**
- **Database Tables**: Each DocType corresponds to a database table
- **Form Definitions**: They auto-generate forms, lists, and reports
- **API Blueprints**: REST endpoints are created automatically
- **Data Models**: Complete with validation rules and business logic

### DocType vs Traditional Models

| Traditional ORM | Frappe DocType |
|------------------|----------------|
| Code-defined schemas | Metadata-driven definitions |
| Static structure | Dynamic, runtime-configurable |
| Developer-only changes | Admin-configurable |
| Manual form creation | Auto-generated UI |
| Custom API endpoints | Automatic REST APIs |

## DocType Architecture

### Core Structure

```json
{
    "doctype": "DocType",
    "name": "Library Member",
    "module": "Library Management", 
    "fields": [
        {
            "fieldname": "member_name",
            "fieldtype": "Data",
            "label": "Full Name",
            "reqd": 1
        }
    ],
    "permissions": [
        {
            "role": "Librarian",
            "read": 1,
            "write": 1
        }
    ]
}
```

### Key Components

**1. Metadata Definition** (`doctype.json`)
```python
# Auto-generated types from frappe/core/doctype/doctype/doctype.py:100-150
class DocType(Document):
    # Basic Properties
    module: DF.Link                    # Which app module this belongs to
    is_submittable: DF.Check          # Can documents be submitted/cancelled
    istable: DF.Check                 # Is this a child table
    issingle: DF.Check                # Single document (like settings)
    is_tree: DF.Check                 # Hierarchical structure
    
    # Form Behavior
    allow_copy: DF.Check              # Allow document duplication
    allow_rename: DF.Check            # Allow renaming documents
    allow_import: DF.Check            # Allow data import
    hide_toolbar: DF.Check            # Hide standard toolbar
    
    # Fields and Structure
    fields: DF.Table[DocField]        # List of document fields
    permissions: DF.Table[DocPerm]    # Permission rules
```

**2. Controller Logic** (`.py` file)
```python
# apps/library_management/doctype/library_member/library_member.py
class LibraryMember(Document):
    def validate(self):
        """Custom validation logic"""
        self.validate_email()
    
    def before_save(self):
        """Pre-save processing"""
        self.full_name = f"{self.first_name} {self.last_name}"
    
    def on_submit(self):
        """Post-submission actions"""
        self.send_welcome_email()
```

**3. Client-Side Behavior** (`.js` file)
```javascript
// library_member.js
frappe.ui.form.on('Library Member', {
    refresh: function(frm) {
        // Form customizations
        frm.add_custom_button('Send Email', function() {
            // Custom actions
        });
    }
});
```

## Field Types & Properties

### Standard Field Types

**Data Fields**
```json
{
    "fieldname": "email",
    "fieldtype": "Data",
    "label": "Email Address",
    "options": "Email",
    "reqd": 1,
    "unique": 1
}
```

**Complete Field Type Reference:**

| Field Type | Description | Example Use Case |
|------------|-------------|------------------|
| **Data** | Single line text (140 chars) | Name, Email, Phone |
| **Small Text** | Multi-line text (250 chars) | Short Description |
| **Text** | Long text (unlimited) | Comments, Notes |
| **Text Editor** | Rich text with formatting | Article Content |
| **Int** | Integer numbers | Quantity, Age |
| **Float** | Decimal numbers | Price, Weight |
| **Currency** | Formatted currency | Amount, Balance |
| **Date** | Date picker | Birth Date, Due Date |
| **Datetime** | Date and time | Created At, Event Time |
| **Time** | Time picker | Working Hours |
| **Check** | Boolean checkbox | Is Active, Published |
| **Select** | Dropdown options | Status, Priority |
| **Link** | Reference to another DocType | Customer, Item |
| **Dynamic Link** | Variable reference | Reference Type/Name |
| **Table** | Child table | Order Items, Contact Details |
| **Attach** | File attachment | Documents, Images |
| **Attach Image** | Image attachment | Photos, Logos |
| **Signature** | Digital signature | Approval Signatures |
| **Color** | Color picker | Theme Colors |
| **Barcode** | Barcode display/scan | Product Codes |
| **Geolocation** | Geographic coordinates | Store Locations |
| **Duration** | Time duration | Task Duration |
| **Rating** | Star rating | Customer Rating |
| **Password** | Encrypted password field | Login Credentials |
| **Code** | Code editor | Custom Scripts |
| **JSON** | JSON data | Configuration Data |

### Field Properties

**Core Properties:**
```json
{
    "fieldname": "customer_name",        // Database column name (required)
    "fieldtype": "Data",                 // Field type (required)  
    "label": "Customer Name",            // Display label
    "reqd": 1,                          // Required field
    "unique": 1,                        // Unique constraint
    "read_only": 1,                     // Read-only field
    "hidden": 1,                        // Hidden from form
    "default": "Active",                // Default value
    "description": "Customer full name", // Help text
    "options": "Customer",              // For Link/Select fields
    "length": 100,                      // Maximum length
    "precision": 2,                     // Decimal places (for Float/Currency)
    "depends_on": "eval:doc.is_customer", // Conditional display
    "mandatory_depends_on": "doc.type=='Customer'", // Conditional required
    "read_only_depends_on": "doc.submitted", // Conditional read-only
    "fetch_from": "customer.customer_name", // Fetch value from linked doc
    "fetch_if_empty": 1,                // Only fetch if empty
    "in_list_view": 1,                  // Show in list view
    "in_standard_filter": 1,            // Add to standard filters
    "in_global_search": 1,              // Include in global search
    "search_index": 1,                  // Database index for search
    "collapsible": 1,                   // Collapsible section
    "allow_bulk_edit": 1,               // Allow bulk operations
    "translatable": 1,                  // Support translations
    "print_hide": 1,                    // Hide in print formats
    "report_hide": 1,                   // Hide in reports
    "width": "200px",                   // Column width
    "columns": 2                        // Grid columns
}
```

**Validation Properties:**
```json
{
    "fieldname": "phone",
    "fieldtype": "Data", 
    "label": "Phone Number",
    "options": "Phone",              // Built-in validation
    "reqd": 1,                       // Required validation
    "unique": 1,                     // Unique validation
    "ignore_user_permissions": 1,     // Skip user permission checks
    "ignore_xss_filter": 1,          // Skip XSS filtering
    "allow_in_quick_entry": 1,       // Show in quick entry
    "no_copy": 1                     // Don't copy when duplicating
}
```

## DocType Creation & Management

### Creating DocTypes via UI

**Step 1: Navigation**
```
Desk → DocType → New
```

**Step 2: Basic Information**
```json
{
    "name": "Library Member",
    "module": "Library Management",
    "description": "Members who can borrow books"
}
```

**Step 3: Field Definition**
- Add fields using the Form Builder
- Configure field properties
- Set permissions and naming rules

### Creating DocTypes Programmatically

**Method 1: JSON Import**
```python
# create_library_member.py
import frappe
import json

doctype_dict = {
    "doctype": "DocType",
    "name": "Library Member", 
    "module": "Library Management",
    "fields": [
        {
            "fieldname": "member_name",
            "fieldtype": "Data", 
            "label": "Member Name",
            "reqd": 1
        },
        {
            "fieldname": "email",
            "fieldtype": "Data",
            "label": "Email",
            "options": "Email",
            "unique": 1
        },
        {
            "fieldname": "phone", 
            "fieldtype": "Data",
            "label": "Phone Number"
        },
        {
            "fieldname": "membership_type",
            "fieldtype": "Select",
            "label": "Membership Type",
            "options": "\nStandard\nPremium\nStudent",
            "default": "Standard"
        }
    ],
    "permissions": [
        {
            "role": "Librarian",
            "read": 1,
            "write": 1,
            "create": 1,
            "delete": 1
        },
        {
            "role": "System Manager", 
            "read": 1,
            "write": 1,
            "create": 1,
            "delete": 1
        }
    ]
}

# Create the DocType
doc = frappe.get_doc(doctype_dict)
doc.insert()
```

**Method 2: Python Builder Pattern**
```python
# Using frappe.new_doc for cleaner syntax
def create_library_member_doctype():
    if frappe.db.exists("DocType", "Library Member"):
        return
    
    doctype = frappe.new_doc("DocType")
    doctype.name = "Library Member"
    doctype.module = "Library Management" 
    doctype.description = "Library member management"
    
    # Add fields
    doctype.append("fields", {
        "fieldname": "member_name",
        "fieldtype": "Data",
        "label": "Member Name", 
        "reqd": 1,
        "in_list_view": 1
    })
    
    doctype.append("fields", {
        "fieldname": "email",
        "fieldtype": "Data", 
        "label": "Email Address",
        "options": "Email",
        "unique": 1,
        "in_list_view": 1
    })
    
    # Add permissions
    doctype.append("permissions", {
        "role": "Librarian",
        "read": 1, "write": 1, "create": 1, "delete": 1
    })
    
    doctype.insert()
    print(f"Created DocType: {doctype.name}")
```

### DocType Configuration Options

**Basic Settings:**
```python
# From doctype.py analysis - Core configuration options
{
    "is_submittable": 0,        # Documents can be submitted/cancelled
    "istable": 0,              # Child table DocType
    "issingle": 0,             # Single document type (like Settings)
    "is_tree": 0,              # Tree/hierarchical structure
    "track_changes": 0,        # Version control
    "track_seen": 0,           # Track who viewed
    "track_views": 0,          # View statistics
    "allow_copy": 1,           # Allow document duplication
    "allow_rename": 1,         # Allow renaming documents
    "allow_import": 1,         # Data import capability
    "allow_auto_repeat": 0,    # Auto-repeat functionality
    "custom": 0,               # Custom vs standard DocType
    "beta": 0,                 # Beta feature flag
    "editable_grid": 1,        # Grid editing capability
    "quick_entry": 0,          # Quick entry dialog
    "hide_toolbar": 0,         # Hide standard toolbar
    "make_attachments_public": 0, // Public file access
    "max_attachments": 0       # Attachment limit
}
```

**Naming Configuration:**
```python
{
    "naming_rule": "By fieldname",     # Naming method
    "autoname": "field:member_id",     # Auto-naming pattern
    "title_field": "member_name",      # Title field for display
    "search_fields": "member_name,email", // Search fields
    "sort_field": "modified",          # Default sort
    "sort_order": "DESC"               # Sort order
}
```

## Document Lifecycle

### Standard Lifecycle States

Based on analysis of `frappe/model/docstatus.py`:

```python
# Document status constants
class DocStatus:
    draft = 0      # Draft - can be modified
    submitted = 1  # Submitted - read-only, can be cancelled  
    cancelled = 2  # Cancelled - read-only
```

### Lifecycle Hooks

**Server-Side Hooks** (in `.py` controller):
```python
class LibraryMember(Document):
    def validate(self):
        """Called before saving (both insert and update)"""
        self.validate_email_format()
        self.calculate_membership_fee()
    
    def before_validate(self):
        """Called before validate"""
        self.standardize_phone_number()
    
    def after_validate(self):
        """Called after validate"""
        pass
    
    def before_insert(self):
        """Called before first save"""
        self.set_member_id()
    
    def after_insert(self):
        """Called after first save"""
        self.send_welcome_email()
    
    def before_save(self):
        """Called before every save"""
        self.set_full_name()
    
    def after_save(self):
        """Called after every save"""
        self.update_member_statistics()
    
    def on_update(self):
        """Called after save (same as after_save)"""
        pass
    
    def on_submit(self):
        """Called when document is submitted"""
        self.create_membership_card()
    
    def on_cancel(self):
        """Called when document is cancelled"""
        self.cancel_membership_benefits()
    
    def before_rename(self, old_name, new_name, merge=False):
        """Called before renaming"""
        pass
    
    def after_rename(self, old_name, new_name, merge=False):
        """Called after renaming"""
        self.update_references()
    
    def on_trash(self):
        """Called before deletion"""
        self.cleanup_related_data()
    
    def after_delete(self):
        """Called after deletion"""
        pass
```

**Complete Lifecycle Flow:**
```
Document Creation:
before_validate → validate → after_validate → before_insert → 
before_save → after_save → after_insert

Document Update:
before_validate → validate → after_validate → before_save → 
after_save → on_update

Document Submission:
before_validate → validate → after_validate → before_save → 
before_submit → on_submit → after_save

Document Cancellation:
before_validate → validate → after_validate → before_save → 
before_cancel → on_cancel → after_save

Document Deletion:
on_trash → after_delete
```

## Relationships & Links

### Link Fields

**Basic Link Field:**
```json
{
    "fieldname": "customer",
    "fieldtype": "Link", 
    "label": "Customer",
    "options": "Customer",
    "reqd": 1
}
```

**Link with Filters:**
```json
{
    "fieldname": "active_customer",
    "fieldtype": "Link",
    "label": "Active Customer", 
    "options": "Customer",
    "link_filters": {
        "status": "Active",
        "customer_group": ["!=", "Blocked"]
    }
}
```

### Dynamic Links

**Dynamic Link Pattern:**
```json
// Reference Type field
{
    "fieldname": "reference_type",
    "fieldtype": "Select", 
    "label": "Reference Type",
    "options": "Customer\nSupplier\nEmployee"
}

// Dynamic Link field
{
    "fieldname": "reference_name",
    "fieldtype": "Dynamic Link",
    "label": "Reference", 
    "options": "reference_type"
}
```

### Child Tables

**Parent DocType Configuration:**
```json
{
    "fieldname": "contact_details", 
    "fieldtype": "Table",
    "label": "Contact Details",
    "options": "Contact Detail"  // Child DocType name
}
```

**Child DocType Setup:**
```json
{
    "doctype": "DocType",
    "name": "Contact Detail",
    "istable": 1,  // Mark as child table
    "fields": [
        {
            "fieldname": "contact_type",
            "fieldtype": "Select",
            "label": "Contact Type", 
            "options": "Phone\nEmail\nAddress"
        },
        {
            "fieldname": "contact_value",
            "fieldtype": "Data",
            "label": "Contact Value"
        }
    ]
}
```

**Accessing Child Table Data:**
```python
# Server-side access
def validate(self):
    for contact in self.contact_details:
        if contact.contact_type == "Email":
            self.validate_email(contact.contact_value)

# Get child table records
contacts = frappe.get_all("Contact Detail", 
    filters={"parent": self.name},
    fields=["contact_type", "contact_value"]
)
```

### Tree Structure

**Tree DocType Configuration:**
```json
{
    "doctype": "DocType",
    "name": "Department",
    "is_tree": 1,
    "fields": [
        {
            "fieldname": "department_name", 
            "fieldtype": "Data",
            "label": "Department Name"
        },
        {
            "fieldname": "parent_department",
            "fieldtype": "Link",
            "label": "Parent Department",
            "options": "Department"
        }
    ]
}
```

## Naming & Auto-naming

### Naming Rules

Based on analysis of `frappe/model/naming.py:24`:

**Available Naming Rules:**
1. **Set by user** - Manual naming
2. **Autoincrement** - Sequential numbers 
3. **By fieldname** - Use a specific field value
4. **By "Naming Series" field** - Naming series pattern
5. **Expression** - Python expression

### Naming Series Pattern

```python
# From frappe/model/naming.py:23-24
NAMING_SERIES_PATTERN = re.compile(r"^[\w\- \/.#{}]+$", re.UNICODE)
BRACED_PARAMS_PATTERN = re.compile(r"(\{[\w | #]+\})")
```

**Naming Series Examples:**
```python
# Basic series
"LIB-MEM-.#####"          # LIB-MEM-00001, LIB-MEM-00002...

# With year
"LIB-.YYYY.-.#####"       # LIB-2024-00001, LIB-2024-00002...

# With field values
"LIB-{membership_type}-.#####"  # LIB-Premium-00001

# Complex pattern
"LIB-{branch_code}-.YYYY.-.MM.-.#####"  # LIB-NYC-2024-03-00001
```

**Naming Series Implementation:**
```python
class LibraryMember(Document):
    def autoname(self):
        """Custom naming logic"""
        if self.membership_type == "Premium":
            self.name = f"PREM-{self.member_name[:3].upper()}-{frappe.generate_hash()[:6]}"
        else:
            # Use default naming series
            pass
    
    def validate_name(self):
        """Custom name validation"""
        if not self.name.startswith("LIB-"):
            frappe.throw("Member ID must start with 'LIB-'")
```

### Custom Auto-naming

**Expression-Based Naming:**
```json
{
    "naming_rule": "Expression",
    "autoname": "format:LIB-{member_name}-{###}"
}
```

**Field-Based Naming:**
```json
{
    "naming_rule": "By fieldname", 
    "autoname": "field:member_id"
}
```

**Python Method Naming:**
```python
def autoname(self):
    """Custom naming method in DocType controller"""
    # Custom logic for generating names
    branch_code = frappe.get_value("Branch", self.branch, "branch_code")
    year = frappe.utils.nowdate()[:4]
    sequence = self.get_next_sequence_number(branch_code, year)
    self.name = f"{branch_code}-{year}-{sequence:05d}"
```

## Permissions & Security

### Permission Structure

**Role-Based Permissions:**
```json
{
    "role": "Librarian",
    "read": 1,           // Can read documents
    "write": 1,          // Can modify documents  
    "create": 1,         // Can create new documents
    "delete": 1,         // Can delete documents
    "submit": 1,         // Can submit documents (if submittable)
    "cancel": 1,         // Can cancel documents (if submittable)
    "amend": 1,          // Can amend cancelled documents
    "report": 1,         // Can access reports
    "export": 1,         // Can export data
    "import": 1,         // Can import data
    "print": 1,          // Can print documents
    "email": 1,          // Can email documents
    "share": 1,          // Can share documents
    "set_user_permissions": 1  // Can set user-specific permissions
}
```

### User Permissions

**Document-Level Restrictions:**
```python
# Restrict user to specific branches
frappe.get_doc({
    "doctype": "User Permission",
    "user": "librarian@library.com",
    "allow": "Branch", 
    "for_value": "Main Branch",
    "applicable_for": "Library Member"
}).insert()

# Apply in queries automatically
members = frappe.get_all("Library Member", 
    fields=["name", "member_name"]
)  # Only shows members from Main Branch for this user
```

### Field-Level Permissions

**Conditional Field Access:**
```json
{
    "fieldname": "salary",
    "fieldtype": "Currency",
    "label": "Salary",
    "read_only": 1,
    "permlevel": 1  // Higher permission level required
}
```

**Permission Levels:**
```json
{
    "role": "HR Manager",
    "permlevel": 1,    // Can access level 1 fields
    "read": 1,
    "write": 1
}
```

## Advanced Features

### Single DocTypes

**Configuration:**
```json
{
    "doctype": "DocType",
    "name": "Library Settings",
    "issingle": 1,       // Single document type
    "fields": [
        {
            "fieldname": "max_books_per_member",
            "fieldtype": "Int", 
            "label": "Max Books Per Member",
            "default": 5
        }
    ]
}
```

**Usage:**
```python
# Always use doctype name as document name for singles
settings = frappe.get_doc("Library Settings", "Library Settings")
max_books = settings.max_books_per_member

# Or use get_single
settings = frappe.get_single("Library Settings")
```

### Virtual DocTypes

**For External Data Sources:**
```python
class APIDataDocType(Document):
    """Virtual DocType for external API data"""
    
    def db_insert(self):
        """Override to prevent database insertion"""
        pass
    
    def load_from_db(self):
        """Load data from external source"""
        # Fetch from external API
        api_data = self.fetch_from_api()
        self.update(api_data)
    
    @staticmethod
    def get_list(args):
        """Override list method"""
        # Return data from external source
        return fetch_list_from_api(args)
```

### Custom Form Scripts

**Advanced Form Customization:**
```javascript
frappe.ui.form.on('Library Member', {
    setup: function(frm) {
        // Form setup logic
        frm.set_query("branch", function() {
            return {
                "filters": {
                    "is_active": 1
                }
            };
        });
    },
    
    refresh: function(frm) {
        // Dynamic buttons based on document state
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__('Generate ID Card'), function() {
                frappe.call({
                    method: 'library_management.api.generate_id_card',
                    args: {'member': frm.doc.name},
                    callback: function(r) {
                        if (r.message) {
                            window.open(r.message.file_url);
                        }
                    }
                });
            });
        }
    },
    
    membership_type: function(frm) {
        // Auto-calculate fees based on membership type
        if (frm.doc.membership_type) {
            frappe.call({
                method: 'library_management.api.get_membership_fee',
                args: {'membership_type': frm.doc.membership_type},
                callback: function(r) {
                    frm.set_value('membership_fee', r.message);
                }
            });
        }
    },
    
    validate: function(frm) {
        // Client-side validation
        if (frm.doc.email && !frappe.utils.validate_type(frm.doc.email, 'email')) {
            frappe.msgprint(__('Please enter a valid email address'));
            validated = false;
        }
    }
});

// Child table events
frappe.ui.form.on('Contact Detail', {
    contact_details_add: function(frm, cdt, cdn) {
        // Called when new row is added
        var row = locals[cdt][cdn];
        row.contact_type = 'Phone'; // Set default
    },
    
    contact_type: function(frm, cdt, cdn) {
        // Called when contact_type changes
        var row = locals[cdt][cdn];
        if (row.contact_type === 'Email') {
            // Set email validation
            frappe.model.set_df_property(cdt, 'contact_value', 'options', 'Email');
        }
    }
});
```

## Best Practices

### Design Principles

**1. Naming Conventions**
```python
# DocType names: Title Case with spaces
"Library Member", "Book Issue", "Fine Payment"

# Field names: snake_case
"member_name", "issue_date", "return_date"

# Module names: Title Case with spaces  
"Library Management", "Asset Management"
```

**2. Field Organization**
```json
{
    "fields": [
        // Group related fields with section breaks
        {"fieldtype": "Section Break", "label": "Basic Information"},
        {"fieldname": "member_name", "fieldtype": "Data"},
        {"fieldname": "email", "fieldtype": "Data"},
        
        {"fieldtype": "Column Break"},
        {"fieldname": "phone", "fieldtype": "Data"},
        {"fieldname": "address", "fieldtype": "Small Text"},
        
        {"fieldtype": "Section Break", "label": "Membership Details"},
        {"fieldname": "membership_type", "fieldtype": "Select"},
        {"fieldname": "membership_date", "fieldtype": "Date"}
    ]
}
```

**3. Validation Strategy**
```python
def validate(self):
    """Layered validation approach"""
    self.validate_basic_data()      # Data format validation
    self.validate_business_rules()  # Business logic validation  
    self.validate_dependencies()    # Cross-document validation
    
def validate_basic_data(self):
    """Basic data validation"""
    if self.email and "@" not in self.email:
        frappe.throw("Invalid email format")
    
def validate_business_rules(self): 
    """Business logic validation"""
    if self.membership_type == "Student" and not self.student_id:
        frappe.throw("Student ID required for student membership")
        
def validate_dependencies(self):
    """Cross-document validation"""
    existing_member = frappe.db.exists("Library Member", {
        "email": self.email,
        "name": ("!=", self.name)
    })
    if existing_member:
        frappe.throw("Member with this email already exists")
```

**4. Performance Optimization**
```python
# Use db methods for simple queries instead of get_doc
count = frappe.db.count("Library Member", {"status": "Active"})

# Batch operations for bulk updates
frappe.db.sql("""
    UPDATE `tabLibrary Member` 
    SET status = 'Inactive' 
    WHERE last_visit < %s
""", (cutoff_date,))

# Index frequently queried fields
{
    "fieldname": "member_email",
    "search_index": 1,  # Creates database index
    "unique": 1         # Unique constraint
}
```

**5. Security Considerations**
```python
def validate(self):
    # Sanitize user input
    self.member_name = frappe.utils.sanitize_html(self.member_name)
    
    # Permission checks
    if not frappe.has_permission("Branch", "read", self.branch):
        frappe.throw("You don't have access to this branch")
    
def has_permission(doc, user, permission_type):
    """Custom permission logic"""
    if user_has_role(user, "Library Admin"):
        return True
    
    # Allow users to only see their own branch members
    user_branch = frappe.get_value("User", user, "branch")
    return doc.branch == user_branch
```

### Common Patterns

**1. Status Management**
```python
class LibraryMember(Document):
    def on_update_after_submit(self):
        """Allow status updates after submission"""
        self.update_member_status()
    
    def update_member_status(self):
        """Smart status updates based on activity"""
        last_issue = frappe.db.get_value("Book Issue", 
            {"member": self.name}, "issue_date", order_by="issue_date desc")
        
        if last_issue and date_diff(nowdate(), last_issue) > 90:
            self.db_set("status", "Inactive")
```

**2. Calculated Fields**
```python
def before_save(self):
    """Calculate derived fields"""
    self.age = date_diff(nowdate(), self.birth_date) // 365
    self.full_name = f"{self.first_name} {self.last_name}".strip()
    
    # Membership fee calculation
    if self.membership_type == "Premium":
        self.membership_fee = 100
    elif self.membership_type == "Student":
        self.membership_fee = 25
    else:
        self.membership_fee = 50
```

**3. Integration Hooks**
```python
def after_insert(self):
    """Post-creation integrations"""
    # Create user account
    if self.email and not frappe.db.exists("User", self.email):
        self.create_user_account()
    
    # Send welcome email
    self.send_welcome_notification()
    
    # Update statistics
    self.update_membership_statistics()
```

---

**Next Steps**: Learn about [Server-Side Development](03-server-side-development.md) to implement advanced business logic and integrations.