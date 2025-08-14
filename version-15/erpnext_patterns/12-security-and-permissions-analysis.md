# ERPNext Security and Permissions Analysis

## Table of Contents

1. [Overview](#overview)
2. [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
3. [User Permissions System](#user-permissions-system)
4. [Website Portal Security](#website-portal-security)
5. [Permission Query Filtering](#permission-query-filtering)
6. [Document-Level Security](#document-level-security)
7. [API Security Patterns](#api-security-patterns)
8. [Employee Access Control](#employee-access-control)
9. [Regional Permission Management](#regional-permission-management)
10. [Security Validation Patterns](#security-validation-patterns)
11. [Custom Permission Implementation](#custom-permission-implementation)
12. [Security Best Practices](#security-best-practices)
13. [Troubleshooting Security Issues](#troubleshooting-security-issues)

## Overview

ERPNext implements a comprehensive multi-layered security architecture that combines role-based access control, user-specific permissions, document-level security, and specialized portal access patterns. This analysis examines production-proven security patterns from ERPNext's codebase to provide comprehensive guidance for enterprise security implementation.

### Core Security Principles

1. **Defense in Depth**: Multiple security layers for comprehensive protection
2. **Principle of Least Privilege**: Users receive minimum necessary permissions
3. **Role-Based Access**: Systematic role-based permission management
4. **Data Segregation**: Multi-company and user-specific data isolation
5. **Portal Security**: Secure external stakeholder access
6. **Audit Trail**: Comprehensive activity logging and monitoring

## Role-Based Access Control (RBAC)

ERPNext implements sophisticated role-based access control with hierarchical permission inheritance and granular permission management.

### Basic Permission Checking Pattern
*Reference: Multiple files showing `frappe.has_permission()` usage*

```python
# Standard permission checking pattern
@frappe.whitelist()
def close_or_unclose_sales_orders(names, status):
    """Example of role-based permission checking"""
    if not frappe.has_permission("Sales Order", "write"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    names = json.loads(names)
    for name in names:
        so = frappe.get_doc("Sales Order", name)
        so.update_status(status)

# Advanced permission checking with specific operations
@frappe.whitelist()
def make_raw_material_request(items, company, sales_order, project=None):
    """Permission check for complex operations"""
    if not frappe.has_permission("Sales Order", "write"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Additional business logic permission checks
    if not frappe.has_permission("Material Request", "create"):
        frappe.throw(_("Cannot create Material Request"), frappe.PermissionError)
    
    if isinstance(items, str):
        items = frappe._dict(json.loads(items))
    
    # Proceed with operation
    return create_material_requests(items, company, sales_order, project)
```

### Permission Validation Framework

```python
def validate_permissions_for_operation(operation_config):
    """Comprehensive permission validation framework"""
    required_permissions = operation_config.get("required_permissions", [])
    
    for perm_config in required_permissions:
        doctype = perm_config["doctype"]
        ptype = perm_config["permission_type"]  # read, write, create, delete, submit, etc.
        
        if not frappe.has_permission(doctype, ptype):
            frappe.throw(
                _("You do not have {0} permission for {1}").format(ptype, doctype),
                frappe.PermissionError
            )
    
    # Check additional business rule permissions
    business_rules = operation_config.get("business_rule_checks", [])
    for rule_check in business_rules:
        if not rule_check["validator"]():
            frappe.throw(rule_check["error_message"], frappe.PermissionError)

def check_multi_doctype_permissions(doctypes_permissions):
    """Check permissions across multiple doctypes atomically"""
    validation_results = {}
    
    for doctype, ptype in doctypes_permissions.items():
        has_perm = frappe.has_permission(doctype, ptype)
        validation_results[doctype] = has_perm
        
        if not has_perm:
            return False, validation_results
    
    return True, validation_results

# Usage example
operation_permissions = {
    "Sales Order": "write",
    "Material Request": "create", 
    "Purchase Order": "create",
    "Item": "read"
}

success, results = check_multi_doctype_permissions(operation_permissions)
if not success:
    frappe.throw(_("Insufficient permissions for multi-step operation"))
```

### Role Management Patterns

```python
def setup_role_hierarchy():
    """Setup hierarchical role structure"""
    role_hierarchy = {
        "System Manager": {
            "inherits_from": [],
            "permissions": ["all"]
        },
        "Sales Manager": {
            "inherits_from": ["Sales User"],
            "additional_permissions": [
                {"doctype": "Sales Order", "permissions": ["create", "read", "write", "submit", "cancel"]},
                {"doctype": "Customer", "permissions": ["create", "read", "write"]},
                {"doctype": "Quotation", "permissions": ["create", "read", "write", "submit"]}
            ]
        },
        "Sales User": {
            "inherits_from": [],
            "permissions": [
                {"doctype": "Sales Order", "permissions": ["read", "write"]},
                {"doctype": "Customer", "permissions": ["read"]},
                {"doctype": "Quotation", "permissions": ["create", "read", "write"]}
            ]
        }
    }
    
    for role_name, config in role_hierarchy.items():
        create_role_with_permissions(role_name, config)

def create_role_with_permissions(role_name, config):
    """Create role with defined permissions"""
    if not frappe.db.exists("Role", role_name):
        role_doc = frappe.get_doc({
            "doctype": "Role",
            "role_name": role_name,
            "desk_access": 1
        })
        role_doc.insert()
    
    # Apply permissions for each doctype
    for perm_config in config.get("permissions", []):
        if perm_config == "all":
            continue  # System Manager gets all permissions by default
            
        apply_doctype_permissions(role_name, perm_config)

def apply_doctype_permissions(role_name, perm_config):
    """Apply specific permissions to role for doctype"""
    doctype = perm_config["doctype"]
    permissions = perm_config["permissions"]
    
    perm_doc = frappe.get_doc({
        "doctype": "DocPerm",
        "parent": doctype,
        "role": role_name,
        "read": 1 if "read" in permissions else 0,
        "write": 1 if "write" in permissions else 0,
        "create": 1 if "create" in permissions else 0,
        "delete": 1 if "delete" in permissions else 0,
        "submit": 1 if "submit" in permissions else 0,
        "cancel": 1 if "cancel" in permissions else 0,
        "amend": 1 if "amend" in permissions else 0
    })
    
    perm_doc.insert()
```

## User Permissions System

ERPNext implements granular user permissions that restrict access to specific records within a DocType.

### User Permission Management Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/setup/doctype/employee/employee.py:87-103`*

```python
def update_user_permissions(self):
    """Advanced user permission management from Employee DocType"""
    if not self.has_value_changed("user_id") and not self.has_value_changed("create_user_permission"):
        return

    # Check if current user has permission to modify user permissions
    if not has_permission("User Permission", ptype="write", raise_exception=False):
        return

    # Check existing user permission
    employee_user_permission_exists = frappe.db.exists(
        "User Permission", 
        {"allow": "Employee", "for_value": self.name, "user": self.user_id}
    )

    if employee_user_permission_exists and not self.create_user_permission:
        # Remove permissions when no longer needed
        remove_user_permission("Employee", self.name, self.user_id)
        remove_user_permission("Company", self.company, self.user_id)
    elif not employee_user_permission_exists and self.create_user_permission:
        # Add permissions when required
        add_user_permission("Employee", self.name, self.user_id)
        add_user_permission("Company", self.company, self.user_id)
```

### Advanced User Permission Patterns

```python
def setup_hierarchical_user_permissions(user, hierarchy_config):
    """Setup hierarchical user permissions based on organizational structure"""
    
    # Clear existing permissions
    clear_user_permissions(user)
    
    # Apply base permissions
    base_permissions = hierarchy_config.get("base_permissions", [])
    for perm in base_permissions:
        add_user_permission(perm["doctype"], perm["value"], user)
    
    # Apply hierarchical permissions based on reports_to structure
    if hierarchy_config.get("enable_hierarchy"):
        subordinate_permissions = get_subordinate_permissions(user)
        for perm in subordinate_permissions:
            add_user_permission(perm["doctype"], perm["value"], user)

def get_subordinate_permissions(user):
    """Get permissions for records related to subordinates"""
    subordinate_permissions = []
    
    # Get employee record for user
    employee = frappe.db.get_value("Employee", {"user_id": user}, "name")
    if not employee:
        return subordinate_permissions
    
    # Get all subordinate employees in hierarchy
    subordinates = get_employee_hierarchy(employee)
    
    # Create permissions for subordinate-related records
    permission_doctypes = ["Sales Order", "Purchase Order", "Leave Application", "Expense Claim"]
    
    for doctype in permission_doctypes:
        subordinate_records = get_subordinate_records(doctype, subordinates)
        for record in subordinate_records:
            subordinate_permissions.append({
                "doctype": doctype,
                "value": record
            })
    
    return subordinate_permissions

def get_employee_hierarchy(employee):
    """Get complete employee hierarchy below given employee"""
    from frappe.utils.nestedset import get_descendants_of
    
    return get_descendants_of("Employee", employee)

def setup_territory_based_permissions(user, territory):
    """Setup permissions based on sales territory"""
    territory_config = {
        "Customer": get_customers_in_territory(territory),
        "Sales Order": get_sales_orders_in_territory(territory),
        "Quotation": get_quotations_in_territory(territory),
        "Lead": get_leads_in_territory(territory)
    }
    
    for doctype, records in territory_config.items():
        for record in records:
            add_user_permission(doctype, record, user)

def setup_multi_company_permissions(user, companies):
    """Setup multi-company access permissions"""
    for company in companies:
        # Add company permission
        add_user_permission("Company", company, user)
        
        # Add related permissions
        company_related_permissions = {
            "Warehouse": get_warehouses_for_company(company),
            "Cost Center": get_cost_centers_for_company(company), 
            "Account": get_accounts_for_company(company)
        }
        
        for doctype, records in company_related_permissions.items():
            for record in records:
                add_user_permission(doctype, record, user)
```

### Dynamic Permission Assignment

```python
def assign_dynamic_permissions_on_document_create(doc, method):
    """Assign permissions dynamically when documents are created"""
    
    # Get permission assignment rules for this doctype
    assignment_rules = get_permission_assignment_rules(doc.doctype)
    
    for rule in assignment_rules:
        if evaluate_rule_condition(rule, doc):
            assign_permissions_per_rule(rule, doc)

def get_permission_assignment_rules(doctype):
    """Get permission assignment rules for doctype"""
    return frappe.get_all(
        "Permission Assignment Rule",
        filters={"doctype": doctype, "enabled": 1},
        fields=["name", "condition", "assign_to_field", "permission_type"]
    )

def evaluate_rule_condition(rule, doc):
    """Evaluate whether rule condition is met"""
    if not rule.get("condition"):
        return True
    
    # Use safe eval to check condition
    try:
        condition_result = frappe.safe_eval(
            rule.condition,
            {"doc": doc, "frappe": frappe}
        )
        return bool(condition_result)
    except Exception:
        frappe.log_error("Permission rule evaluation failed")
        return False

def assign_permissions_per_rule(rule, doc):
    """Assign permissions based on rule configuration"""
    assign_to_field = rule.get("assign_to_field")
    if not assign_to_field or not doc.get(assign_to_field):
        return
    
    user = doc.get(assign_to_field)
    permission_type = rule.get("permission_type", "read")
    
    # Create user permission
    add_user_permission(doc.doctype, doc.name, user)
    
    # Log permission assignment
    frappe.log_info(
        f"Assigned {permission_type} permission for {doc.doctype} {doc.name} to {user}"
    )
```

## Website Portal Security

ERPNext implements sophisticated portal security for customer and supplier access.

### Portal User Access Control
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/controllers/website_list_for_contact.py:224-241`*

```python
def get_customers_suppliers(doctype, user):
    """Get customers/suppliers accessible to portal user"""
    customers = []
    suppliers = []
    meta = frappe.get_meta(doctype)

    customer_field_name = get_customer_field_name(doctype)
    has_customer_field = meta.has_field(customer_field_name)
    has_supplier_field = meta.has_field("supplier")

    # Check if user has Customer or Supplier role
    if has_common(["Supplier", "Customer"], frappe.get_roles(user)):
        suppliers = get_parents_for_user("Supplier")
        customers = get_parents_for_user("Customer")
    elif frappe.has_permission(doctype, "read", user=user):
        # Full access for internal users
        customer_list = frappe.get_list("Customer")
        customers = suppliers = [customer.name for customer in customer_list]

    return customers if has_customer_field else None, suppliers if has_supplier_field else None

def get_parents_for_user(parenttype: str) -> list[str]:
    """Get parent records (Customer/Supplier) for portal user"""
    portal_user = frappe.qb.DocType("Portal User")

    return (
        frappe.qb.from_(portal_user)
        .select(portal_user.parent)
        .where(portal_user.user == frappe.session.user)
        .where(portal_user.parenttype == parenttype)
    ).run(pluck="name")
```

### Website Permission Validation
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/controllers/website_list_for_contact.py:255-264`*

```python
def has_website_permission(doc, ptype, user, verbose=False):
    """Check website permissions for portal users"""
    doctype = doc.doctype
    customers, suppliers = get_customers_suppliers(doctype, user)
    
    if customers:
        return frappe.db.exists(doctype, get_customer_filter(doc, customers))
    elif suppliers:
        fieldname = "suppliers" if doctype == "Request for Quotation" else "supplier"
        return frappe.db.exists(doctype, {"name": doc.name, fieldname: ["in", suppliers]})
    else:
        return False

def get_customer_filter(doc, customers):
    """Build customer-specific filter for document access"""
    doctype = doc.doctype
    filters = frappe._dict()
    filters.name = doc.name
    filters[get_customer_field_name(doctype)] = ["in", customers]
    
    if doctype == "Quotation":
        filters.quotation_to = "Customer"
    
    return filters
```

### Portal Transaction List Security
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/controllers/website_list_for_contact.py:66-124`*

```python
def get_transaction_list(
    doctype,
    txt=None,
    filters=None,
    limit_start=0,
    limit_page_length=20,
    order_by="creation desc",
    custom=False,
):
    """Secure transaction list for portal users"""
    user = frappe.session.user
    ignore_permissions = False

    if not filters:
        filters = {}

    filters["docstatus"] = ["<", "2"] if doctype in ["Supplier Quotation", "Purchase Invoice"] else 1

    if (user != "Guest" and is_website_user()) or doctype == "Request for Quotation":
        parties_doctype = "Request for Quotation Supplier" if doctype == "Request for Quotation" else doctype
        
        # Find party association for this contact
        customers, suppliers = get_customers_suppliers(parties_doctype, user)

        if customers:
            if doctype == "Quotation":
                filters["quotation_to"] = "Customer"
                filters["party_name"] = ["in", customers]
            else:
                filters["customer"] = ["in", customers]
        elif suppliers:
            filters["supplier"] = ["in", suppliers]
        elif not custom:
            return []  # No access if no party association

        # Portal users get filtered access, not full permissions
        ignore_permissions = True

    transactions = get_list_for_transactions(
        doctype,
        txt,
        filters,
        limit_start,
        limit_page_length,
        fields="name",
        ignore_permissions=ignore_permissions,
        order_by=order_by,
    )

    return post_process(doctype, transactions) if not custom else transactions
```

### Advanced Portal Security Patterns

```python
def setup_customer_portal_security(customer_name):
    """Setup comprehensive portal security for customer"""
    
    # Get all contacts for customer
    contacts = frappe.get_all(
        "Contact",
        filters={"name": ["in", frappe.get_all(
            "Dynamic Link",
            filters={"link_doctype": "Customer", "link_name": customer_name},
            pluck="parent"
        )]},
        fields=["name", "email_id"]
    )
    
    for contact in contacts:
        if contact.email_id:
            setup_portal_user_security(contact.email_id, "Customer", customer_name)

def setup_portal_user_security(email, party_type, party_name):
    """Setup portal user with appropriate security constraints"""
    
    # Create or update user
    user = get_or_create_portal_user(email, party_type)
    
    # Setup portal user link
    create_portal_user_link(user.name, party_type, party_name)
    
    # Apply security restrictions
    apply_portal_security_restrictions(user.name, party_type, party_name)

def apply_portal_security_restrictions(user, party_type, party_name):
    """Apply comprehensive portal security restrictions"""
    
    # Restrict to specific company if applicable
    if party_type == "Customer":
        customer_doc = frappe.get_doc("Customer", party_name)
        if customer_doc.default_company:
            add_user_permission("Company", customer_doc.default_company, user)
    
    # Add document-specific restrictions
    portal_restrictions = {
        "Customer": {
            "allowed_doctypes": ["Sales Order", "Sales Invoice", "Delivery Note", "Quotation"],
            "filters": {"customer": party_name}
        },
        "Supplier": {
            "allowed_doctypes": ["Purchase Order", "Purchase Invoice", "Purchase Receipt", "Supplier Quotation"],
            "filters": {"supplier": party_name}
        }
    }
    
    restriction_config = portal_restrictions.get(party_type, {})
    for doctype in restriction_config.get("allowed_doctypes", []):
        setup_doctype_portal_restrictions(user, doctype, restriction_config["filters"])

def setup_doctype_portal_restrictions(user, doctype, filters):
    """Setup doctype-specific portal restrictions"""
    
    # Create permission entry with filters
    perm_doc = frappe.get_doc({
        "doctype": "Custom DocPerm",
        "parent": doctype,
        "role": "Customer" if "customer" in filters else "Supplier",
        "read": 1,
        "write": 0,  # Portal users typically can't edit
        "create": 0,
        "delete": 0,
        "report": 1,
        "export": 1,
        "print": 1,
        "email": 1
    })
    
    perm_doc.insert(ignore_if_duplicate=True)
```

## Permission Query Filtering

ERPNext implements dynamic permission filtering at the query level for performance and security.

### Employee Query with Permissions
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/controllers/queries.py:22-69`*

```python
@frappe.whitelist()
@frappe.validate_and_sanitize_search_inputs
def employee_query(
    doctype,
    txt,
    searchfield,
    start,
    page_len,
    filters,
    reference_doctype: str | None = None,
    ignore_user_permissions: bool = False,
):
    """Secure employee query with permission filtering"""
    doctype = "Employee"
    conditions = []
    fields = get_fields(doctype, ["name", "employee_name"])
    ignore_permissions = False

    # Check if permissions can be ignored for this context
    if reference_doctype and ignore_user_permissions:
        ignore_permissions = has_ignored_field(reference_doctype, doctype) and has_permission(
            doctype,
            ptype="select" if frappe.only_has_select_perm(doctype) else "read",
        )

    search_conditions = " or ".join([f"{field} like %(txt)s" for field in fields])
    
    # Apply match conditions based on user permissions
    mcond = "" if ignore_permissions else get_match_cond(doctype)

    return frappe.db.sql(
        """select {fields} from `tabEmployee`
        where status in ('Active', 'Suspended')
            and docstatus < 2
            and ({key} like %(txt)s or {search_conditions})
            {fcond} {mcond}
        order by
            (case when locate(%(_txt)s, name) > 0 then locate(%(_txt)s, name) else 99999 end),
            (case when locate(%(_txt)s, employee_name) > 0 then locate(%(_txt)s, employee_name) else 99999 end),
            idx desc,
            name, employee_name
        limit %(page_len)s offset %(start)s""".format(
            **{
                "fields": ", ".join(fields),
                "key": searchfield,
                "fcond": get_filters_cond(doctype, filters, conditions),
                "mcond": mcond,
                "search_conditions": search_conditions,
            }
        ),
        {"txt": "%%%s%%" % txt, "_txt": txt.replace("%", ""), "start": start, "page_len": page_len},
    )
```

### Dynamic Permission Query Building

```python
def build_permission_aware_query(doctype, fields, filters=None, user=None):
    """Build query with dynamic permission filtering"""
    
    if not user:
        user = frappe.session.user
    
    # Base query
    query = frappe.qb.from_(frappe.qb.DocType(doctype))
    
    # Apply field selection
    if isinstance(fields, str):
        fields = [fields]
    
    for field in fields:
        query = query.select(field)
    
    # Apply base filters
    if filters:
        for field, condition in filters.items():
            if isinstance(condition, list):
                query = query.where(frappe.qb.Field(field).isin(condition))
            else:
                query = query.where(frappe.qb.Field(field) == condition)
    
    # Apply permission filters
    permission_filters = get_user_permission_filters(doctype, user)
    for perm_filter in permission_filters:
        query = query.where(perm_filter)
    
    return query

def get_user_permission_filters(doctype, user):
    """Get permission-based query filters for user"""
    permission_filters = []
    
    # Get user permissions for this doctype
    user_permissions = frappe.get_all(
        "User Permission",
        filters={"user": user, "allow": doctype},
        fields=["for_value"]
    )
    
    if user_permissions:
        allowed_values = [up.for_value for up in user_permissions]
        permission_filters.append(
            frappe.qb.Field("name").isin(allowed_values)
        )
    
    # Get linked document permissions
    linked_permissions = get_linked_document_permissions(doctype, user)
    for field, allowed_values in linked_permissions.items():
        if allowed_values:
            permission_filters.append(
                frappe.qb.Field(field).isin(allowed_values)
            )
    
    return permission_filters

def get_linked_document_permissions(doctype, user):
    """Get permissions for linked documents"""
    linked_permissions = {}
    
    meta = frappe.get_meta(doctype)
    
    for field in meta.fields:
        if field.fieldtype == "Link":
            # Get user permissions for linked doctype
            linked_permissions_data = frappe.get_all(
                "User Permission",
                filters={"user": user, "allow": field.options},
                fields=["for_value"]
            )
            
            if linked_permissions_data:
                linked_permissions[field.fieldname] = [
                    lp.for_value for lp in linked_permissions_data
                ]
    
    return linked_permissions
```

### Advanced Query Permission Patterns

```python
def apply_territory_based_filtering(query, doctype, user):
    """Apply territory-based filtering to queries"""
    
    # Get user's territory permissions
    user_territories = frappe.get_all(
        "User Permission",
        filters={"user": user, "allow": "Territory"},
        pluck="for_value"
    )
    
    if not user_territories:
        return query  # No territory restrictions
    
    meta = frappe.get_meta(doctype)
    
    # Check if doctype has territory field
    if meta.has_field("territory"):
        query = query.where(
            frappe.qb.Field("territory").isin(user_territories)
        )
    
    # Check for customer-based territory filtering
    if meta.has_field("customer"):
        customers_in_territory = frappe.get_all(
            "Customer",
            filters={"territory": ["in", user_territories]},
            pluck="name"
        )
        
        if customers_in_territory:
            query = query.where(
                frappe.qb.Field("customer").isin(customers_in_territory)
            )
    
    return query

def apply_company_based_filtering(query, doctype, user):
    """Apply company-based filtering to queries"""
    
    user_companies = frappe.get_all(
        "User Permission",
        filters={"user": user, "allow": "Company"},
        pluck="for_value"
    )
    
    if not user_companies:
        return query  # No company restrictions
    
    meta = frappe.get_meta(doctype)
    
    if meta.has_field("company"):
        query = query.where(
            frappe.qb.Field("company").isin(user_companies)
        )
    
    return query

def apply_hierarchical_filtering(query, doctype, user):
    """Apply hierarchical filtering based on employee hierarchy"""
    
    # Get employee record for user
    employee = frappe.db.get_value("Employee", {"user_id": user}, "name")
    if not employee:
        return query  # No hierarchy restrictions for non-employees
    
    # Get subordinate employees
    from frappe.utils.nestedset import get_descendants_of
    subordinates = get_descendants_of("Employee", employee)
    subordinates.append(employee)  # Include self
    
    meta = frappe.get_meta(doctype)
    
    # Apply filtering based on document ownership
    owner_fields = ["owner", "created_by", "employee", "sales_person"]
    
    for field in owner_fields:
        if meta.has_field(field):
            if field == "owner":
                # Get users for subordinate employees
                subordinate_users = frappe.get_all(
                    "Employee",
                    filters={"name": ["in", subordinates]},
                    pluck="user_id"
                )
                subordinate_users = [u for u in subordinate_users if u]  # Remove None values
                
                if subordinate_users:
                    query = query.where(
                        frappe.qb.Field(field).isin(subordinate_users)
                    )
            else:
                query = query.where(
                    frappe.qb.Field(field).isin(subordinates)
                )
            break  # Apply only first matching field
    
    return query
```

## Document-Level Security

### Document Validation Security

```python
def validate_document_security(doc, method):
    """Comprehensive document-level security validation"""
    
    # Check basic permissions
    validate_basic_permissions(doc)
    
    # Check field-level security
    validate_field_level_security(doc)
    
    # Check business rule security
    validate_business_rule_security(doc)
    
    # Check data integrity security
    validate_data_integrity_security(doc)

def validate_basic_permissions(doc):
    """Validate basic CRUD permissions for document"""
    
    if doc.is_new():
        required_permission = "create"
    elif doc.has_value_changed() and not doc.is_new():
        required_permission = "write"
    else:
        required_permission = "read"
    
    if not frappe.has_permission(doc.doctype, required_permission):
        frappe.throw(
            _("You don't have {0} permission for {1}").format(
                required_permission, doc.doctype
            ),
            frappe.PermissionError
        )

def validate_field_level_security(doc):
    """Validate field-level security restrictions"""
    
    meta = frappe.get_meta(doc.doctype)
    
    for field in meta.fields:
        # Check if field has restricted access
        if field.get("restricted_to_roles"):
            user_roles = set(frappe.get_roles())
            allowed_roles = set(field.restricted_to_roles.split(","))
            
            if not user_roles.intersection(allowed_roles):
                if doc.has_value_changed(field.fieldname):
                    frappe.throw(
                        _("You don't have permission to modify {0}").format(field.label),
                        frappe.PermissionError
                    )

def validate_business_rule_security(doc):
    """Validate business rule-based security"""
    
    # Example: Validate expense claim amount limits
    if doc.doctype == "Expense Claim":
        validate_expense_claim_limits(doc)
    
    # Example: Validate sales order discount limits
    if doc.doctype == "Sales Order":
        validate_discount_limits(doc)
    
    # Example: Validate stock transaction authority
    if doc.doctype in ["Stock Entry", "Stock Reconciliation"]:
        validate_stock_transaction_authority(doc)

def validate_expense_claim_limits(doc):
    """Validate expense claim amount limits based on user role"""
    
    user_roles = frappe.get_roles()
    max_amount = 0
    
    # Define role-based limits
    role_limits = {
        "Employee": 10000,
        "Manager": 50000,
        "Finance Manager": 100000,
        "System Manager": float("inf")
    }
    
    # Get maximum allowed amount for user
    for role in user_roles:
        if role in role_limits:
            max_amount = max(max_amount, role_limits[role])
    
    if doc.total_sanctioned_amount > max_amount:
        frappe.throw(
            _("You can only approve expense claims up to {0}").format(
                frappe.format_value(max_amount, {"fieldtype": "Currency"})
            ),
            frappe.PermissionError
        )

def validate_discount_limits(doc):
    """Validate discount limits for sales orders"""
    
    user_discount_limit = get_user_discount_limit(frappe.session.user)
    
    for item in doc.items:
        if item.discount_percentage > user_discount_limit:
            frappe.throw(
                _("You cannot apply discount more than {0}% on item {1}").format(
                    user_discount_limit, item.item_code
                ),
                frappe.PermissionError
            )

def get_user_discount_limit(user):
    """Get maximum discount limit for user"""
    
    # Check if user has specific discount limit
    user_limit = frappe.db.get_value("User", user, "max_discount_percentage")
    if user_limit:
        return user_limit
    
    # Check role-based discount limits
    user_roles = frappe.get_roles(user)
    role_limits = {
        "Sales User": 5,
        "Sales Manager": 15,
        "Sales Director": 25,
        "System Manager": 100
    }
    
    max_limit = 0
    for role in user_roles:
        if role in role_limits:
            max_limit = max(max_limit, role_limits[role])
    
    return max_limit
```

## API Security Patterns

### API Permission Validation

```python
@frappe.whitelist()
def secure_api_endpoint(data):
    """Example of secure API endpoint implementation"""
    
    # Input validation
    if not data:
        frappe.throw(_("No data provided"), frappe.ValidationError)
    
    # Parse and validate JSON data
    try:
        if isinstance(data, str):
            data = frappe.parse_json(data)
    except ValueError:
        frappe.throw(_("Invalid JSON data"), frappe.ValidationError)
    
    # Permission validation
    required_doctype = data.get("doctype")
    if not required_doctype:
        frappe.throw(_("DocType is required"), frappe.ValidationError)
    
    if not frappe.has_permission(required_doctype, "create"):
        frappe.throw(_("Insufficient permissions"), frappe.PermissionError)
    
    # Additional business logic validation
    validate_api_business_rules(data)
    
    # Process the request
    return process_secure_api_request(data)

def validate_api_business_rules(data):
    """Validate business rules for API requests"""
    
    # Example: Validate company access
    if data.get("company"):
        user_companies = get_user_companies(frappe.session.user)
        if data["company"] not in user_companies:
            frappe.throw(_("You don't have access to this company"), frappe.PermissionError)
    
    # Example: Validate customer access for portal users
    if frappe.utils.is_website_user():
        if data.get("customer"):
            accessible_customers = get_accessible_customers(frappe.session.user)
            if data["customer"] not in accessible_customers:
                frappe.throw(_("You don't have access to this customer"), frappe.PermissionError)

@frappe.whitelist()
def bulk_operation_with_security(operations):
    """Secure bulk operation endpoint"""
    
    if not operations:
        frappe.throw(_("No operations provided"))
    
    if isinstance(operations, str):
        operations = frappe.parse_json(operations)
    
    # Validate permissions for all operations before processing
    for operation in operations:
        doctype = operation.get("doctype")
        operation_type = operation.get("operation")  # create, update, delete
        
        if not frappe.has_permission(doctype, operation_type):
            frappe.throw(
                _("Insufficient permissions for {0} operation on {1}").format(
                    operation_type, doctype
                ),
                frappe.PermissionError
            )
    
    # Process operations with transaction management
    results = []
    for operation in operations:
        try:
            frappe.db.savepoint("bulk_operation")
            result = process_single_operation(operation)
            results.append({"status": "success", "result": result})
        except Exception as e:
            frappe.db.rollback(save_point="bulk_operation")
            results.append({"status": "error", "error": str(e)})
    
    return results
```

### Rate Limiting and Throttling

```python
def implement_api_rate_limiting():
    """Implement API rate limiting for security"""
    
    user = frappe.session.user
    endpoint = frappe.request.path
    
    # Get rate limit for user/endpoint combination
    rate_limit = get_api_rate_limit(user, endpoint)
    
    # Check current usage
    current_usage = get_current_api_usage(user, endpoint)
    
    if current_usage >= rate_limit:
        frappe.throw(
            _("Rate limit exceeded. Try again later."),
            frappe.RateLimitExceededError
        )
    
    # Increment usage counter
    increment_api_usage(user, endpoint)

def get_api_rate_limit(user, endpoint):
    """Get rate limit for user and endpoint"""
    
    # Check user-specific limits
    user_limit = frappe.db.get_value(
        "User API Limit",
        {"user": user, "endpoint": endpoint},
        "requests_per_hour"
    )
    
    if user_limit:
        return user_limit
    
    # Check role-based limits
    user_roles = frappe.get_roles(user)
    role_limits = {
        "System Manager": 10000,
        "User": 1000,
        "Guest": 100
    }
    
    for role in user_roles:
        if role in role_limits:
            return role_limits[role]
    
    return 100  # Default limit

def get_current_api_usage(user, endpoint):
    """Get current API usage for user and endpoint"""
    
    # Get usage from last hour
    from_time = frappe.utils.add_to_date(frappe.utils.now(), hours=-1)
    
    usage_count = frappe.db.count(
        "API Usage Log",
        {
            "user": user,
            "endpoint": endpoint,
            "timestamp": [">=", from_time]
        }
    )
    
    return usage_count

def log_api_access():
    """Log API access for security monitoring"""
    
    api_log = frappe.get_doc({
        "doctype": "API Access Log",
        "user": frappe.session.user,
        "endpoint": frappe.request.path,
        "method": frappe.request.method,
        "ip_address": frappe.local.request_ip,
        "user_agent": frappe.request.headers.get("User-Agent"),
        "timestamp": frappe.utils.now(),
        "response_time": getattr(frappe.local, "response_time", 0)
    })
    
    api_log.insert(ignore_permissions=True)
```

## Regional Permission Management

### Regional Permission Setup Pattern
*Reference: `/home/frappe/frappe-bench/apps/erpnext/erpnext/regional/italy/setup.py:480-496`*

```python
def add_permissions():
    """Regional permission setup pattern"""
    doctype = "Import Supplier Invoice"
    
    # Clear existing permissions
    add_permission(doctype, "All", 0)

    # Add role-based permissions systematically
    for role in ("Accounts Manager", "Accounts User", "Purchase User", "Auditor"):
        add_permission(doctype, role, 0)
        update_permission_property(doctype, role, 0, "print", 1)
        update_permission_property(doctype, role, 0, "report", 1)

        # Conditional write access
        if role in ("Accounts Manager", "Accounts User"):
            update_permission_property(doctype, role, 0, "write", 1)
            update_permission_property(doctype, role, 0, "create", 1)

    # Manager-specific permissions
    update_permission_property(doctype, "Accounts Manager", 0, "delete", 1)
    add_permission(doctype, "Accounts Manager", 1)  # Submit permission
    update_permission_property(doctype, "Accounts Manager", 1, "write", 1)
    update_permission_property(doctype, "Accounts Manager", 1, "create", 1)
```

### Systematic Regional Permission Implementation

```python
def setup_regional_permissions(country):
    """Setup comprehensive regional permissions"""
    
    regional_config = get_regional_permission_config(country)
    
    for config in regional_config:
        setup_doctype_regional_permissions(config)

def get_regional_permission_config(country):
    """Get regional permission configuration"""
    
    configs = {
        "Italy": [
            {
                "doctype": "Electronic Invoice Register",
                "roles": {
                    "Accounts Manager": ["read", "write", "create", "delete", "report"],
                    "Accounts User": ["read", "write", "create", "report"],
                    "Auditor": ["read", "report"]
                }
            },
            {
                "doctype": "Import Supplier Invoice", 
                "roles": {
                    "Accounts Manager": ["read", "write", "create", "delete", "submit", "report", "print"],
                    "Accounts User": ["read", "write", "create", "report", "print"],
                    "Purchase User": ["read", "report", "print"],
                    "Auditor": ["read", "report", "print"]
                }
            }
        ],
        "United Arab Emirates": [
            {
                "doctype": "UAE VAT Settings",
                "roles": {
                    "Accounts Manager": ["read", "write", "create"],
                    "Accounts User": ["read", "write", "create"],
                    "System Manager": ["read", "write", "create", "delete"]
                }
            },
            {
                "doctype": "UAE VAT Account",
                "roles": {
                    "Accounts Manager": ["read", "write", "create"],
                    "Accounts User": ["read", "write", "create"],
                    "System Manager": ["read", "write", "create", "delete"]
                }
            }
        ]
    }
    
    return configs.get(country, [])

def setup_doctype_regional_permissions(config):
    """Setup permissions for specific doctype"""
    
    doctype = config["doctype"]
    roles_config = config["roles"]
    
    # Clear existing permissions
    clear_doctype_permissions(doctype)
    
    # Add permissions for each role
    for role, permissions in roles_config.items():
        add_permission(doctype, role, 0)  # Draft permission level
        
        for ptype in permissions:
            if ptype == "submit":
                # Submit permissions need permission level 1
                add_permission(doctype, role, 1)
                update_permission_property(doctype, role, 1, "write", 1)
                update_permission_property(doctype, role, 1, "create", 1)
            else:
                update_permission_property(doctype, role, 0, ptype, 1)

def clear_doctype_permissions(doctype):
    """Clear all existing permissions for doctype"""
    
    frappe.db.delete("DocPerm", {"parent": doctype})
    frappe.db.delete("Custom DocPerm", {"parent": doctype})
```

## Security Best Practices

### Secure Coding Patterns

```python
def implement_security_best_practices():
    """Comprehensive security best practices implementation"""
    
    security_measures = {
        "input_validation": implement_input_validation(),
        "sql_injection_prevention": implement_sql_injection_prevention(),
        "xss_prevention": implement_xss_prevention(),
        "csrf_protection": implement_csrf_protection(),
        "session_management": implement_secure_session_management(),
        "password_security": implement_password_security(),
        "audit_logging": implement_audit_logging()
    }
    
    return security_measures

def implement_input_validation():
    """Implement comprehensive input validation"""
    
    validation_rules = {
        "email": validate_email_security,
        "phone": validate_phone_security,
        "currency": validate_currency_security,
        "date": validate_date_security,
        "integer": validate_integer_security,
        "text": validate_text_security
    }
    
    def validate_input(value, field_type, additional_rules=None):
        """Universal input validation function"""
        
        if not value:
            return True  # Let required field validation handle empty values
        
        # Apply base validation for field type
        if field_type in validation_rules:
            if not validation_rules[field_type](value):
                frappe.throw(_("Invalid {0} format").format(field_type))
        
        # Apply additional validation rules
        if additional_rules:
            for rule in additional_rules:
                if not rule["validator"](value):
                    frappe.throw(_(rule["error_message"]))
        
        return True
    
    return validate_input

def validate_email_security(email):
    """Secure email validation"""
    import re
    
    # Basic format validation
    email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(email_regex, email):
        return False
    
    # Check for suspicious patterns
    suspicious_patterns = [
        r'<script',
        r'javascript:',
        r'vbscript:',
        r'onload=',
        r'onerror='
    ]
    
    for pattern in suspicious_patterns:
        if re.search(pattern, email, re.IGNORECASE):
            return False
    
    return True

def implement_sql_injection_prevention():
    """Implement SQL injection prevention measures"""
    
    def secure_query(query, params=None):
        """Execute query with SQL injection prevention"""
        
        # Validate query doesn't contain direct user input
        if not params and any(char in query for char in ["'", '"', ";", "--"]):
            frappe.log_error("Potential SQL injection attempt", query)
            frappe.throw(_("Invalid query format"))
        
        # Use parameterized queries
        return frappe.db.sql(query, params, as_dict=True)
    
    def validate_table_name(table_name):
        """Validate table name to prevent injection"""
        import re
        
        # Allow only alphanumeric characters, underscores
        if not re.match(r'^[a-zA-Z0-9_]+$', table_name):
            frappe.throw(_("Invalid table name"))
        
        # Check if table exists in schema
        if not frappe.db.table_exists(table_name):
            frappe.throw(_("Table does not exist"))
        
        return True
    
    return {"secure_query": secure_query, "validate_table_name": validate_table_name}

def implement_audit_logging():
    """Implement comprehensive audit logging"""
    
    def log_security_event(event_type, details):
        """Log security-related events"""
        
        audit_log = frappe.get_doc({
            "doctype": "Security Audit Log",
            "event_type": event_type,
            "user": frappe.session.user,
            "ip_address": frappe.local.request_ip,
            "user_agent": getattr(frappe.request, "headers", {}).get("User-Agent", ""),
            "details": frappe.as_json(details),
            "timestamp": frappe.utils.now()
        })
        
        audit_log.insert(ignore_permissions=True)
    
    def log_permission_violation(doctype, operation, reason):
        """Log permission violations"""
        
        log_security_event("Permission Violation", {
            "doctype": doctype,
            "operation": operation,
            "reason": reason,
            "user_roles": frappe.get_roles()
        })
    
    def log_login_attempt(success, reason=None):
        """Log login attempts"""
        
        log_security_event("Login Attempt", {
            "success": success,
            "reason": reason,
            "timestamp": frappe.utils.now()
        })
    
    return {
        "log_security_event": log_security_event,
        "log_permission_violation": log_permission_violation,
        "log_login_attempt": log_login_attempt
    }
```

This comprehensive analysis of ERPNext's security and permissions system provides proven patterns for implementing enterprise-grade security that balances usability with robust protection against unauthorized access and data breaches.