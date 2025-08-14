# Permissions & Security - Complete Guide

> **Comprehensive reference for role-based permissions, user management, and security implementation**

## Table of Contents

- [Permission System Overview](#permission-system-overview)
- [Role-Based Access Control](#role-based-access-control)
- [User Management & Authentication](#user-management--authentication)
- [Document-Level Permissions](#document-level-permissions)
- [Field-Level Security](#field-level-security)
- [User Permissions](#user-permissions)
- [Share Permissions](#share-permissions)
- [Custom Permission Logic](#custom-permission-logic)
- [Security Best Practices](#security-best-practices)
- [Permission Debugging](#permission-debugging)

## Permission System Overview

### Permission Architecture

Based on analysis of `frappe/permissions.py:12-37`, Frappe implements a multi-layered permission system:

```python
# Permission types available in Frappe
rights = (
    "select",    # Query/list documents
    "read",      # View document details
    "write",     # Modify documents
    "create",    # Create new documents
    "delete",    # Delete documents
    "submit",    # Submit submittable documents
    "cancel",    # Cancel submitted documents
    "amend",     # Create amended copies
    "print",     # Print documents
    "email",     # Email documents
    "report",    # Access reports
    "import",    # Import data
    "export",    # Export data
    "share",     # Share documents
)

# Standard system roles
GUEST_ROLE = "Guest"
ALL_USER_ROLE = "All"          # All users including website users
SYSTEM_USER_ROLE = "Desk User" # Users with desk access
ADMIN_ROLE = "Administrator"   # Full system access
```

### Permission Hierarchy

**1. System Level**: Framework and application access
**2. DocType Level**: What operations are allowed on document types
**3. Document Level**: Access to specific documents
**4. Field Level**: Granular field access control

### Permission Check Flow

```python
# From frappe/permissions.py:76-87
@print_has_permission_check_logs
def has_permission(doctype, ptype="read", doc=None, user=None, raise_exception=True):
    """
    Main permission check function:
    1. Check if user has role permissions for DocType
    2. Check user permissions (document-level restrictions)
    3. Check share permissions
    4. Check owner permissions
    5. Apply custom permission queries
    """
```

## Role-Based Access Control

### Role Definition

Based on `frappe/core/doctype/role/role.py:12-80`:

```python
class Role(Document):
    # Standard roles that cannot be modified
    STANDARD_ROLES = ("Administrator", "System Manager", "Script Manager", "All", "Guest")
    
    # Role properties
    role_name: str           # Role identifier
    desk_access: bool        # Access to desk interface
    home_page: str          # Default landing page
    two_factor_auth: bool    # Require 2FA for this role
    disabled: bool           # Disable role
    restrict_to_domain: str  # Domain restriction
```

**Creating Custom Roles**:
```python
# Programmatic role creation
role = frappe.get_doc({
    "doctype": "Role",
    "role_name": "Library Manager",
    "desk_access": 1,
    "home_page": "/desk#workspace/Library"
})
role.insert()

# Assign permissions to role
permission = frappe.get_doc({
    "doctype": "Custom DocPerm",  
    "parent": "Library Book",
    "parenttype": "DocType",
    "role": "Library Manager",
    "read": 1,
    "write": 1,
    "create": 1,
    "delete": 1,
    "submit": 0,
    "cancel": 0,
    "permlevel": 0
})
permission.insert()
```

### DocType Permissions Configuration

**Permission Rules** (from `frappe/core/doctype/docperm/docperm.py:7-36`):

```python
class DocPerm(Document):
    # Permission properties
    role: str           # Role name
    permlevel: int      # Permission level (0, 1, 2, etc.)
    
    # CRUD permissions
    read: bool          # Read access
    write: bool         # Write access  
    create: bool        # Create access
    delete: bool        # Delete access
    
    # Workflow permissions
    submit: bool        # Submit documents
    cancel: bool        # Cancel submitted documents
    amend: bool         # Amend cancelled documents
    
    # Additional permissions
    print: bool         # Print documents
    email: bool         # Email documents
    report: bool        # Access reports
    export: bool        # Export data
    import: bool        # Import data
    share: bool         # Share documents
    
    # Conditional permissions
    if_owner: bool      # Only if user owns document
```

**Permission Level System**:
```json
{
    "doctype": "DocType",
    "name": "Sales Invoice",
    "permissions": [
        {
            "role": "Sales User",
            "permlevel": 0,
            "read": 1, "write": 1, "create": 1
        },
        {
            "role": "Accounts Manager", 
            "permlevel": 1,
            "read": 1, "write": 1
        }
    ],
    "fields": [
        {
            "fieldname": "customer",
            "permlevel": 0  // Accessible to Sales User
        },
        {
            "fieldname": "outstanding_amount",
            "permlevel": 1  // Only Accounts Manager can access
        }
    ]
}
```

### Role Assignment Patterns

```python
class LibraryMember(Document):
    def after_insert(self):
        """Automatically assign role to new members"""
        if self.membership_type == "Premium":
            # Create user account for premium members
            user = frappe.get_doc({
                "doctype": "User",
                "email": self.email,
                "first_name": self.first_name,
                "last_name": self.last_name,
                "user_type": "Website User"
            })
            user.insert(ignore_permissions=True)
            
            # Assign library member role
            user.add_roles("Library Member")
```

## User Management & Authentication

### User Document Structure

Based on analysis of `frappe/core/doctype/user/user.py:54-100`:

```python
class User(Document):
    # Core user properties
    email: str                    # Unique identifier
    first_name: str               # User's first name
    last_name: str               # User's last name
    full_name: str               # Computed full name
    user_type: str               # "System User" or "Website User"
    
    # Authentication
    enabled: bool                 # Account enabled/disabled
    api_key: str                 # API authentication key
    api_secret: str              # API authentication secret
    
    # Security settings
    two_factor_auth: bool        # 2FA enabled
    restrict_ip: str            # IP restrictions
    login_after: datetime       # Account active after
    login_before: datetime      # Account expires before
    
    # Preferences
    language: str               # User language
    time_zone: str             # User timezone
    desk_theme: str            # UI theme preference
    
    # Contact information
    phone: str                 # Phone number
    mobile_no: str            # Mobile number
    birth_date: date          # Birth date
```

### User Creation & Management

**Programmatic User Creation**:
```python
def create_library_user(member_doc):
    """Create user account for library member"""
    
    # Check if user already exists
    if frappe.db.exists("User", member_doc.email):
        return frappe.get_doc("User", member_doc.email)
    
    # Create new user
    user = frappe.get_doc({
        "doctype": "User",
        "email": member_doc.email,
        "first_name": member_doc.first_name,
        "last_name": member_doc.last_name,
        "user_type": "Website User",
        "send_welcome_email": 1,
        
        # Role assignments
        "roles": [
            {"role": "Library Member"},
            {"role": "Blogger"}  # Additional role
        ]
    })
    
    user.insert(ignore_permissions=True)
    
    # Set default password
    from frappe.utils.password import update_password
    update_password(user.name, "welcome123", logout_all_sessions=True)
    
    return user

def update_user_permissions_on_transfer(user, from_branch, to_branch):
    """Update user permissions when transferring between branches"""
    
    # Remove old branch permission
    frappe.db.delete("User Permission", {
        "user": user,
        "allow": "Branch",
        "for_value": from_branch
    })
    
    # Add new branch permission
    frappe.get_doc({
        "doctype": "User Permission",
        "user": user,
        "allow": "Branch",
        "for_value": to_branch,
        "apply_to_all_doctypes": 1
    }).insert(ignore_permissions=True)
    
    # Clear user permission cache
    frappe.cache.hdel("user_permissions", user)
```

### Authentication Methods

**Session-Based Authentication**:
```python
def authenticate_user(email, password):
    """Standard user authentication"""
    
    from frappe.auth import check_password
    from frappe.sessions import Session
    
    # Validate credentials
    user = check_password(email, password)
    if not user:
        frappe.throw(_("Invalid login credentials"))
    
    # Check if user is enabled
    if not frappe.get_value("User", user, "enabled"):
        frappe.throw(_("User account is disabled"))
    
    # Create session
    session = Session(user, resume=False)
    session.start()
    
    return user
```

**API Key Authentication**:
```python
def generate_api_credentials(user):
    """Generate API key/secret for user"""
    
    import secrets
    
    api_key = secrets.token_urlsafe(15)
    api_secret = secrets.token_urlsafe(15)
    
    user_doc = frappe.get_doc("User", user)
    user_doc.api_key = api_key
    user_doc.api_secret = api_secret
    user_doc.save(ignore_permissions=True)
    
    return {
        "api_key": api_key,
        "api_secret": api_secret
    }

# API usage
headers = {
    "Authorization": f"token {api_key}:{api_secret}"
}
```

## Document-Level Permissions

### Permission Check Implementation

```python
# Custom permission logic in document controllers
class SalesInvoice(Document):
    def has_permission(doc, user=None, permission_type="read"):
        """Custom document permission logic"""
        
        user = user or frappe.session.user
        
        # Admin always has access
        if user == "Administrator":
            return True
            
        # Check if user is sales person for this customer
        if permission_type in ["read", "write"]:
            sales_person = frappe.get_value("Customer", doc.customer, "sales_person")
            if sales_person == user:
                return True
        
        # Check territory-based access
        user_territory = frappe.get_value("User", user, "territory")
        invoice_territory = frappe.get_value("Customer", doc.customer, "territory")
        
        if user_territory == invoice_territory:
            return True
        
        # Check role-based access
        if frappe.has_permission("Sales Invoice", permission_type, user=user):
            return True
            
        return False
```

### Owner-Based Permissions

```python
# Restrict access to document owners
{
    "role": "Employee",
    "read": 1,
    "write": 1,
    "if_owner": 1  # Only document owner can access
}

# Check owner permissions in controller
def has_permission(doc, user, permission_type):
    """Allow users to access only their own documents"""
    
    if permission_type in ["read", "write"] and doc.owner == user:
        return True
        
    # Allow managers to access subordinate documents
    if frappe.has_permission("Employee", "write", user=user):
        employee = frappe.get_value("User", user, "employee")
        if employee:
            # Check if user is manager of document owner
            return check_manager_subordinate_relation(employee, doc.owner)
    
    return False
```

## Field-Level Security

### Permission Levels

```python
class EmployeeDocument(Document):
    def setup_field_permissions(self):
        """Configure field-level permissions"""
        
        # Basic fields - accessible to all
        basic_fields = ["employee_name", "department", "designation"]
        for field in basic_fields:
            self.meta.get_field(field).permlevel = 0
        
        # Sensitive fields - HR only
        sensitive_fields = ["salary", "bank_account", "tax_id"]
        for field in sensitive_fields:
            self.meta.get_field(field).permlevel = 1
        
        # Confidential fields - Manager only  
        confidential_fields = ["performance_rating", "disciplinary_actions"]
        for field in confidential_fields:
            self.meta.get_field(field).permlevel = 2

# DocType permission configuration
{
    "permissions": [
        {
            "role": "Employee",
            "permlevel": 0,
            "read": 1, "write": 1
        },
        {
            "role": "HR User", 
            "permlevel": 1,
            "read": 1, "write": 1
        },
        {
            "role": "HR Manager",
            "permlevel": 2, 
            "read": 1, "write": 1
        }
    ]
}
```

### Dynamic Field Permissions

```python
def apply_field_permissions_based_on_status(doc):
    """Apply field permissions based on document status"""
    
    if doc.docstatus == 1:  # Submitted
        # Make most fields read-only
        readonly_fields = ["customer", "items", "taxes"]
        for fieldname in readonly_fields:
            doc.set_df_property(fieldname, "read_only", 1)
        
        # Allow certain fields to be modified by specific roles
        if frappe.has_permission("Sales Invoice", "cancel"):
            doc.set_df_property("remarks", "read_only", 0)
    
    elif doc.workflow_state == "Pending Approval":
        # Only approvers can modify during approval
        if not frappe.has_permission("Sales Invoice", "submit"):
            for field in doc.meta.fields:
                if field.fieldtype not in ["Section Break", "Column Break"]:
                    doc.set_df_property(field.fieldname, "read_only", 1)
```

## User Permissions

### User Permission System

Based on `frappe/core/doctype/user_permission/user_permission.py:14-78`:

```python
class UserPermission(Document):
    user: str                    # Target user
    allow: str                   # DocType to restrict
    for_value: str              # Specific document to allow
    applicable_for: str         # Apply restriction to specific DocType
    apply_to_all_doctypes: bool # Apply to all related DocTypes  
    is_default: bool           # Default selection in filters
    hide_descendants: bool     # Hide child records in tree structures
```

### User Permission Implementation

```python
def setup_branch_based_permissions():
    """Set up branch-based user permissions"""
    
    # Get all branch managers
    branch_managers = frappe.get_all("Employee",
        filters={"designation": "Branch Manager", "status": "Active"},
        fields=["name", "user_id", "branch"]
    )
    
    for manager in branch_managers:
        if not manager.user_id or not manager.branch:
            continue
            
        # Create user permission for branch access
        user_perm = frappe.get_doc({
            "doctype": "User Permission",
            "user": manager.user_id,
            "allow": "Branch",
            "for_value": manager.branch,
            "apply_to_all_doctypes": 1,  # Apply to all DocTypes
            "is_default": 1
        })
        
        user_perm.insert(ignore_permissions=True)
        
        # Additional specific permissions
        specific_perms = [
            {"allow": "Customer", "applicable_for": "Sales Order"},
            {"allow": "Supplier", "applicable_for": "Purchase Order"}
        ]
        
        for perm in specific_perms:
            # Get records for this branch
            records = frappe.get_all(perm["allow"], 
                filters={"branch": manager.branch},
                pluck="name"
            )
            
            for record in records:
                frappe.get_doc({
                    "doctype": "User Permission",
                    "user": manager.user_id,
                    "allow": perm["allow"],
                    "for_value": record,
                    "applicable_for": perm["applicable_for"]
                }).insert(ignore_permissions=True)

def check_user_permission_in_query():
    """Apply user permissions to database queries"""
    
    # User permissions are automatically applied to frappe.get_all()
    # and similar queries
    
    # Example: User can only see customers from their branch
    customers = frappe.get_all("Customer", 
        fields=["name", "customer_name", "branch"]
    )
    # Automatically filtered based on user permissions
    
    # To bypass user permissions (with appropriate role)
    all_customers = frappe.get_all("Customer",
        fields=["name", "customer_name", "branch"],
        ignore_permissions=True
    )
```

## Share Permissions

### Document Sharing System

```python
def share_document_with_users(doctype, docname, users, permissions=None):
    """Share document with specific users"""
    
    import frappe.share
    
    default_permissions = {
        "read": 1,
        "write": 0,
        "share": 0
    }
    permissions = permissions or default_permissions
    
    for user in users:
        frappe.share.add(
            doctype=doctype,
            name=docname, 
            user=user,
            read=permissions.get("read", 0),
            write=permissions.get("write", 0),
            share=permissions.get("share", 0),
            notify=1  # Send notification
        )

def check_share_permissions(doc, user, permission_type):
    """Check if user has share permissions for document"""
    
    shared = frappe.get_all("DocShare",
        filters={
            "share_doctype": doc.doctype,
            "share_name": doc.name,
            "user": user
        },
        fields=["read", "write", "share"]
    )
    
    if not shared:
        return False
        
    share_perm = shared[0]
    
    if permission_type == "read" and share_perm.read:
        return True
    elif permission_type == "write" and share_perm.write:
        return True
    elif permission_type == "share" and share_perm.share:
        return True
        
    return False
```

## Custom Permission Logic

### Permission Query Conditions

```python
# hooks.py - Define permission query conditions
permission_query_conditions = {
    "Sales Order": "sales_order.sales_order_conditions",
    "Customer": "sales_order.customer_conditions"
}

# sales_order.py - Permission query implementation
def sales_order_conditions(user):
    """Add conditions to limit Sales Orders based on user permissions"""
    
    if not user:
        user = frappe.session.user
        
    # Admin can see all
    if user == "Administrator":
        return ""
        
    # Sales managers can see orders from their territory
    if "Sales Manager" in frappe.get_roles(user):
        user_territory = frappe.get_value("User", user, "territory")
        if user_territory:
            return f"(`tabSales Order`.territory = '{user_territory}')"
    
    # Sales users can see their own orders
    if "Sales User" in frappe.get_roles(user):
        return f"(`tabSales Order`.owner = '{user}')"
        
    # Default: no access
    return "1=0"

def customer_conditions(user):
    """Limit customers based on sales person assignment"""
    
    employee = frappe.get_value("User", user, "employee")
    if not employee:
        return "1=0"
        
    # Check if user is assigned as sales person for customers
    return f"""(`tabCustomer`.name in (
        SELECT DISTINCT customer 
        FROM `tabSales Person` sp
        JOIN `tabSales Team` st ON sp.name = st.sales_person
        WHERE sp.employee = '{employee}'
    ))"""
```

### Custom Permission Methods

```python
# hooks.py - Custom permission methods
has_permission = {
    "Project": "project.project.has_permission",
    "Task": "project.task.has_permission"
}

# project.py - Custom permission implementation
def has_permission(doc, user=None, permission_type="read"):
    """Custom permission logic for projects"""
    
    user = user or frappe.session.user
    
    if user == "Administrator":
        return True
        
    # Project managers have full access
    if user in (doc.project_manager, doc.project_owner):
        return True
        
    # Team members have read access
    if permission_type == "read":
        team_members = [d.user for d in doc.project_users if d.user]
        if user in team_members:
            return True
    
    # Department-based access
    user_dept = frappe.get_value("User", user, "department")
    if user_dept == doc.department:
        return permission_type in ["read", "write"]
        
    # Check if user has tasks assigned in this project
    if frappe.db.exists("Task", {"project": doc.name, "assigned_to": user}):
        return permission_type == "read"
        
    return False
```

## Security Best Practices

### Input Validation & Sanitization

```python
def validate_and_sanitize_input(doc):
    """Comprehensive input validation"""
    
    # HTML sanitization for text fields
    html_fields = ["description", "remarks", "comments"]
    for fieldname in html_fields:
        if hasattr(doc, fieldname) and doc.get(fieldname):
            doc.set(fieldname, frappe.utils.sanitize_html(doc.get(fieldname)))
    
    # Email validation
    if hasattr(doc, "email") and doc.email:
        if not frappe.utils.validate_email_address(doc.email):
            frappe.throw(_("Invalid email address"))
    
    # Phone number validation
    if hasattr(doc, "phone") and doc.phone:
        if not re.match(r'^[\d\-\+\(\)\s]+$', doc.phone):
            frappe.throw(_("Invalid phone number format"))
    
    # Prevent script injection
    script_fields = doc.get_text_fields()
    for field in script_fields:
        value = doc.get(field)
        if value and ('<script' in value.lower() or 'javascript:' in value.lower()):
            frappe.throw(_("Script tags are not allowed"))
```

### SQL Injection Prevention

```python
def safe_database_queries():
    """Examples of safe database queries"""
    
    # ❌ Vulnerable to SQL injection
    # user_input = frappe.form_dict.get("search")
    # results = frappe.db.sql(f"SELECT * FROM tabCustomer WHERE customer_name LIKE '%{user_input}%'")
    
    # ✅ Safe parameterized query
    user_input = frappe.form_dict.get("search")
    results = frappe.db.sql("""
        SELECT name, customer_name, customer_group
        FROM `tabCustomer` 
        WHERE customer_name LIKE %s
    """, (f"%{user_input}%",), as_dict=True)
    
    # ✅ Using query builder (automatically parameterized)
    from frappe.query_builder import DocType
    Customer = DocType("Customer")
    
    results = (
        frappe.qb.from_(Customer)
        .select(Customer.name, Customer.customer_name)
        .where(Customer.customer_name.like(f"%{user_input}%"))
    ).run(as_dict=True)
```

### CSRF Protection

```python
# CSRF protection is automatic for unsafe HTTP methods
# Manual CSRF validation when needed
def validate_csrf_token():
    """Manual CSRF token validation"""
    
    if frappe.request.method in ["POST", "PUT", "DELETE"]:
        token = frappe.get_request_header("X-Frappe-CSRF-Token")
        if not token:
            token = frappe.form_dict.get("csrf_token")
        
        session_token = frappe.session.data.csrf_token
        if not token or token != session_token:
            frappe.throw(_("Invalid CSRF Token"), frappe.CSRFTokenError)
```

### Password Security

```python
def implement_password_policies():
    """Implement strong password policies"""
    
    # In User DocType validation
    def validate_password_strength(self):
        if self.new_password:
            password = self.new_password
            
            # Minimum length
            if len(password) < 8:
                frappe.throw(_("Password must be at least 8 characters long"))
            
            # Character requirements
            if not re.search(r"[A-Z]", password):
                frappe.throw(_("Password must contain at least one uppercase letter"))
            
            if not re.search(r"[a-z]", password):
                frappe.throw(_("Password must contain at least one lowercase letter"))
            
            if not re.search(r"\d", password):
                frappe.throw(_("Password must contain at least one number"))
            
            if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", password):
                frappe.throw(_("Password must contain at least one special character"))
            
            # Check against common passwords
            common_passwords = ["password", "123456", "admin", "welcome"]
            if password.lower() in common_passwords:
                frappe.throw(_("Please choose a stronger password"))
```

## Permission Debugging

### Debug Permission Checks

```python
def debug_permission_check(doctype, doc=None, user=None):
    """Debug permission check with detailed logging"""
    
    user = user or frappe.session.user
    
    debug_info = {
        "user": user,
        "doctype": doctype,
        "document": doc.name if doc else None,
        "user_roles": frappe.get_roles(user),
        "checks_performed": []
    }
    
    # Check role permissions
    role_perms = frappe.get_all("DocPerm",
        filters={"parent": doctype, "role": ["in", debug_info["user_roles"]]},
        fields=["role", "read", "write", "create", "delete", "permlevel"]
    )
    debug_info["role_permissions"] = role_perms
    
    # Check user permissions
    user_perms = frappe.get_all("User Permission",
        filters={"user": user},
        fields=["allow", "for_value", "applicable_for"]
    )
    debug_info["user_permissions"] = user_perms
    
    # Check share permissions
    if doc:
        share_perms = frappe.get_all("DocShare",
            filters={"share_doctype": doctype, "share_name": doc.name, "user": user},
            fields=["read", "write", "share"]
        )
        debug_info["share_permissions"] = share_perms
    
    return debug_info

# Usage
debug_info = debug_permission_check("Sales Invoice", invoice_doc, "user@example.com")
print(frappe.as_json(debug_info, indent=2))
```

### Permission Error Handling

Based on `frappe/exceptions.py` analysis:

```python
# Common permission-related exceptions
class PermissionError(ValidationError):
    """Raised when user doesn't have required permissions"""
    pass

class CSRFTokenError(ValidationError):
    """Raised when CSRF token is invalid or missing"""
    pass

class AuthenticationError(ValidationError):
    """Raised when authentication fails"""
    pass

# Exception handling in controllers
def handle_permission_errors():
    """Proper permission error handling"""
    
    try:
        # Perform operation
        doc = frappe.get_doc("Sales Invoice", invoice_name)
        doc.submit()
        
    except frappe.PermissionError as e:
        # Log security event
        frappe.log_error(
            message=f"Permission denied for user {frappe.session.user} on {doc.doctype} {doc.name}",
            title="Permission Violation"
        )
        
        # Return user-friendly message
        frappe.throw(_("You don't have permission to perform this action"))
        
    except frappe.AuthenticationError:
        # Redirect to login
        frappe.local.response["type"] = "redirect"
        frappe.local.response["location"] = "/login"
        
    except Exception as e:
        # Log unexpected errors
        frappe.log_error()
        frappe.throw(_("An unexpected error occurred"))
```

### Performance Optimization

```python
def optimize_permission_checks():
    """Optimize permission checking performance"""
    
    # Cache permission results
    @frappe.cache(ttl=300)  # Cache for 5 minutes
    def get_user_permissions(user):
        return frappe.get_all("User Permission",
            filters={"user": user},
            fields=["allow", "for_value", "applicable_for"]
        )
    
    # Batch permission checks
    def check_multiple_documents(doctype, doc_names, permission_type="read"):
        """Check permissions for multiple documents at once"""
        
        results = {}
        
        # Pre-load user data
        user_roles = frappe.get_roles()
        user_perms = get_user_permissions(frappe.session.user)
        
        for doc_name in doc_names:
            # Use cached data for faster checks
            results[doc_name] = frappe.has_permission(
                doctype, permission_type, doc_name,
                user_roles=user_roles,
                user_permissions=user_perms
            )
        
        return results
```

---

**Next Steps**: Continue with [Testing Framework](07-testing-framework.md) to learn about unit testing, integration testing, and automated testing strategies for Frappe applications.