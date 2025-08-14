# API & Integrations - Complete Guide

> **Comprehensive reference for REST API development, authentication, webhooks, and third-party integrations**

## Table of Contents

- [REST API Architecture](#rest-api-architecture)
- [Authentication & Authorization](#authentication--authorization)
- [API Development Patterns](#api-development-patterns)
- [Webhook Implementation](#webhook-implementation)
- [Third-Party Integrations](#third-party-integrations)
- [Data Import/Export Mechanisms](#data-importexport-mechanisms)
- [OAuth & Social Login](#oauth--social-login)
- [Rate Limiting & Security](#rate-limiting--security)
- [API Testing & Documentation](#api-testing--documentation)
- [Advanced Integration Patterns](#advanced-integration-patterns)

## REST API Architecture

### API Versioning System

Based on analysis of `frappe/api/__init__.py:15-81`, Frappe implements versioned APIs:

```python
class ApiVersion(str, Enum):
    V1 = "v1"
    V2 = "v2"

# API routing structure
API_URL_MAP = Map([
    # V1 routes (legacy and default)
    Submount("/api", v1_rules),
    Submount(f"/api/{ApiVersion.V1.value}", v1_rules),
    Submount(f"/api/{ApiVersion.V2.value}", v2_rules),
])
```

**API URL Patterns**:
```
/api/method/{methodname}           # Call whitelisted methods
/api/resource/{doctype}            # Query documents  
/api/resource/{doctype}/{name}     # Specific document operations
/api/v1/...                       # Version 1 endpoints
/api/v2/...                       # Version 2 endpoints
```

### Standard REST Endpoints

**Document CRUD Operations** (from `frappe/api/v1.py`):

```python
# GET /api/resource/DocType - List documents
def document_list(doctype: str):
    frappe.form_dict.setdefault("limit_page_length", 20)
    return frappe.call(frappe.client.get_list, doctype, **frappe.form_dict)

# GET /api/resource/DocType/name - Read document
def read_doc(doctype: str, name: str):
    doc = frappe.get_doc(doctype, name)
    if not doc.has_permission("read"):
        raise frappe.PermissionError
    doc.apply_fieldlevel_read_permissions()
    return doc

# POST /api/resource/DocType - Create document
def create_doc(doctype: str):
    data = get_request_form_data()
    data.pop("doctype", None)
    return frappe.new_doc(doctype, **data).insert()

# PUT /api/resource/DocType/name - Update document
def update_doc(doctype: str, name: str):
    data = get_request_form_data()
    doc = frappe.get_doc(doctype, name, for_update=True)
    doc.update(data)
    doc.save()
    return doc

# DELETE /api/resource/DocType/name - Delete document
def delete_doc(doctype: str, name: str):
    frappe.delete_doc(doctype, name, ignore_missing=False)
    frappe.response.http_status_code = 202
    return "ok"
```

### Query Parameters

**Filtering and Pagination**:
```javascript
// Get filtered list
GET /api/resource/Customer?filters=[["status", "=", "Active"]]&fields=["name", "customer_name"]&limit_page_length=50

// Complex filters
GET /api/resource/Item?filters=[["item_group", "in", ["Products", "Services"]]]&or_filters=[["is_stock_item", "=", 1]]

// Ordering
GET /api/resource/Sales Invoice?order_by=creation desc&limit_start=0&limit_page_length=20
```

**JavaScript API Usage**:
```javascript
// Using frappe.call for API requests
frappe.call({
    method: 'frappe.client.get_list',
    args: {
        doctype: 'Customer',
        filters: {'status': 'Active'},
        fields: ['name', 'customer_name', 'email'],
        order_by: 'creation desc',
        limit_page_length: 50
    },
    callback: function(r) {
        console.log(r.message); // List of customers
    }
});

// Using fetch API directly
fetch('/api/resource/Customer', {
    headers: {
        'X-Frappe-CSRF-Token': frappe.csrf_token
    }
})
.then(response => response.json())
.then(data => console.log(data.data));
```

## Authentication & Authorization

### Authentication System

Based on analysis of `frappe/auth.py:35-99`, Frappe uses session-based authentication:

```python
class HTTPRequest:
    def __init__(self):
        self.set_request_ip()      # Track client IP
        self.set_cookies()         # Load session cookies
        self.set_session()         # Authenticate user
        self.set_lang()           # Set request language  
        self.validate_csrf_token() # CSRF protection
        
    def validate_csrf_token(self):
        # CSRF validation for unsafe HTTP methods
        if (frappe.request.method not in UNSAFE_HTTP_METHODS 
            or frappe.conf.ignore_csrf
            or not frappe.session
            or (frappe.get_request_header("X-Frappe-CSRF-Token") == saved_token)):
            return
        frappe.throw(_("Invalid Request"), frappe.CSRFTokenError)
```

### API Authentication Methods

**1. Session-Based Authentication (Default)**:
```python
# Login endpoint
@frappe.whitelist(allow_guest=True)
def login(usr, pwd):
    try:
        login_manager = frappe.auth.LoginManager()
        login_manager.authenticate(usr, pwd)
        login_manager.post_login()
        
        return {
            "message": "Logged In",
            "user": frappe.session.user,
            "full_name": frappe.utils.get_fullname(frappe.session.user)
        }
    except frappe.AuthenticationError:
        frappe.throw(_("Invalid login credentials"))
```

**2. Token-Based Authentication**:
```python
# Generate API key/secret for users
def generate_api_key(user):
    api_key = frappe.generate_hash(length=15)
    api_secret = frappe.generate_hash(length=15)
    
    user_doc = frappe.get_doc("User", user)
    user_doc.api_key = api_key
    user_doc.api_secret = api_secret
    user_doc.save()
    
    return {"api_key": api_key, "api_secret": api_secret}

# Usage in API requests
headers = {
    "Authorization": f"token {api_key}:{api_secret}"
}
```

**3. OAuth 2.0 Integration**:
```python
# OAuth client configuration (from integrations/doctype/oauth_client)
{
    "client_name": "My App",
    "client_id": "generated_client_id", 
    "client_secret": "generated_client_secret",
    "grant_types": ["authorization_code", "refresh_token"],
    "redirect_uris": ["https://myapp.com/callback"],
    "scopes": ["read", "write"]
}
```

### Permission System Integration

**API Permissions** (from `frappe/api/v2.py:21-24`):
```python
PERMISSION_MAP = {
    "GET": "read",
    "POST": "write",
}

def check_api_permissions(doctype, method):
    permission_type = PERMISSION_MAP.get(method, "read")
    if not frappe.has_permission(doctype, permission_type):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
```

## API Development Patterns

### Custom API Endpoints

**Whitelisted Method Pattern**:
```python
# apps/library_management/library_management/api.py
import frappe
from frappe import _

@frappe.whitelist()
def get_available_books(category=None, limit=20):
    """Get list of available books by category"""
    
    # Permission check
    if not frappe.has_permission("Library Book", "read"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Build filters
    filters = {"status": "Available"}
    if category:
        filters["category"] = category
    
    # Query books
    books = frappe.get_all("Library Book",
        filters=filters,
        fields=["name", "title", "author", "isbn", "category"],
        order_by="title",
        limit_page_length=limit
    )
    
    return {
        "success": True,
        "books": books,
        "total": len(books)
    }

@frappe.whitelist(methods=["POST"])
def issue_book_to_member(book, member, issue_date=None):
    """Issue a book to a library member"""
    
    # Validation
    if not frappe.has_permission("Book Issue", "create"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Business logic validation
    book_doc = frappe.get_doc("Library Book", book)
    if book_doc.status != "Available":
        frappe.throw(_("Book is not available"))
    
    member_doc = frappe.get_doc("Library Member", member)
    if member_doc.status != "Active":
        frappe.throw(_("Member is not active"))
    
    # Create book issue
    issue_doc = frappe.get_doc({
        "doctype": "Book Issue",
        "book": book,
        "member": member,
        "issue_date": issue_date or frappe.utils.nowdate(),
        "status": "Issued"
    })
    
    issue_doc.insert()
    issue_doc.submit()
    
    return {
        "success": True,
        "issue_id": issue_doc.name,
        "message": _("Book issued successfully")
    }

@frappe.whitelist()
@frappe.rate_limit(limit=100, window=3600)  # 100 requests per hour
def search_books(query, limit=50):
    """Search books with rate limiting"""
    
    if len(query) < 3:
        frappe.throw(_("Search query must be at least 3 characters"))
    
    # Sanitize input
    query = frappe.db.escape(f"%{query}%")
    
    # Search query
    results = frappe.db.sql("""
        SELECT name, title, author, isbn
        FROM `tabLibrary Book` 
        WHERE status = 'Available'
        AND (title LIKE %s OR author LIKE %s OR isbn LIKE %s)
        ORDER BY title
        LIMIT %s
    """, (query, query, query, limit), as_dict=True)
    
    return {
        "success": True,
        "results": results,
        "query": query.strip('%')
    }
```

### Batch API Operations

**Bulk Operations Pattern**:
```python
@frappe.whitelist(methods=["POST"])
def bulk_update_book_status(book_updates):
    """Update multiple books' status in one API call"""
    
    if not frappe.has_permission("Library Book", "write"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    results = []
    errors = []
    
    for update in book_updates:
        try:
            book_id = update.get("name")
            new_status = update.get("status")
            
            # Validate
            if not book_id or not new_status:
                raise ValueError("Book name and status required")
            
            # Update book
            book_doc = frappe.get_doc("Library Book", book_id)
            book_doc.status = new_status
            book_doc.save()
            
            results.append({
                "name": book_id,
                "status": "updated",
                "new_status": new_status
            })
            
        except Exception as e:
            errors.append({
                "name": update.get("name", "Unknown"),
                "error": str(e)
            })
    
    # Commit successful updates
    frappe.db.commit()
    
    return {
        "success": True,
        "updated": len(results),
        "errors": len(errors),
        "results": results,
        "error_details": errors
    }
```

### File Upload APIs

**File Handling Pattern** (based on `frappe/core/api/file.py`):
```python
@frappe.whitelist()
def upload_member_document(member):
    """Upload document for library member"""
    
    if not frappe.has_permission("Library Member", "write", member):
        frappe.throw(_("Not permitted"), frappe.PermissionError)
    
    # Handle file upload
    if not frappe.request.files:
        frappe.throw(_("No file attached"))
    
    file_obj = frappe.request.files.get('file')
    if not file_obj:
        frappe.throw(_("File is required"))
    
    # Validate file type
    allowed_types = ['application/pdf', 'image/jpeg', 'image/png']
    if file_obj.content_type not in allowed_types:
        frappe.throw(_("Only PDF and image files are allowed"))
    
    # Validate file size (5MB limit)
    if len(file_obj.read()) > 5 * 1024 * 1024:
        frappe.throw(_("File size cannot exceed 5MB"))
    
    file_obj.seek(0)  # Reset file pointer
    
    # Save file
    from frappe.utils.file_manager import save_file
    
    file_doc = save_file(
        filename=file_obj.filename,
        content=file_obj.read(),
        dt="Library Member",
        dn=member,
        is_private=1
    )
    
    return {
        "success": True,
        "file_url": file_doc.file_url,
        "file_name": file_doc.file_name
    }
```

## Webhook Implementation

### Webhook Configuration

Based on analysis of `frappe/integrations/doctype/webhook/webhook.py`:

```python
class Webhook(Document):
    # Webhook events
    EVENTS = [
        "after_insert", "on_update", "on_submit", 
        "on_cancel", "on_trash", "on_update_after_submit", "on_change"
    ]
    
    def validate(self):
        self.validate_docevent()     # Event validation
        self.validate_condition()    # Condition syntax
        self.validate_request_url()  # URL validation
        self.validate_request_body() # Body template validation
```

**Webhook DocType Configuration**:
```json
{
    "doctype": "Webhook",
    "webhook_doctype": "Library Member",
    "webhook_docevent": "after_insert",
    "request_url": "https://api.example.com/webhook",
    "request_method": "POST",
    "request_structure": "JSON",
    "condition": "doc.membership_type == 'Premium'",
    "webhook_json": {
        "event": "member_created",
        "member_id": "{{ doc.name }}",
        "member_name": "{{ doc.member_name }}",
        "email": "{{ doc.email }}",
        "membership_type": "{{ doc.membership_type }}"
    }
}
```

### Webhook Security

**Webhook Signature Verification**:
```python
WEBHOOK_SECRET_HEADER = "X-Frappe-Webhook-Signature"

def generate_webhook_signature(data, secret):
    """Generate HMAC signature for webhook verification"""
    import hmac
    import hashlib
    
    signature = hmac.new(
        secret.encode('utf-8'),
        data.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    
    return f"sha256={signature}"

def verify_webhook_signature(request_body, signature, secret):
    """Verify webhook signature"""
    expected_signature = generate_webhook_signature(request_body, secret)
    return hmac.compare_digest(signature, expected_signature)
```

### Custom Webhook Processing

**Advanced Webhook Handler**:
```python
# apps/library_management/library_management/webhooks.py

def process_member_webhook(doc, method):
    """Custom webhook processing for member events"""
    
    # Get all active webhooks for this DocType and event
    webhooks = frappe.get_all("Webhook",
        filters={
            "webhook_doctype": doc.doctype,
            "webhook_docevent": method,
            "enabled": 1
        },
        fields=["name", "request_url", "condition", "webhook_json"]
    )
    
    for webhook in webhooks:
        # Check condition if specified
        if webhook.condition:
            try:
                condition_result = frappe.safe_eval(
                    webhook.condition, 
                    eval_locals={"doc": doc}
                )
                if not condition_result:
                    continue
            except Exception:
                frappe.log_error(f"Webhook condition error: {webhook.name}")
                continue
        
        # Enqueue webhook call
        frappe.enqueue(
            "library_management.webhooks.send_webhook",
            queue="short",
            webhook_name=webhook.name,
            doc_data=doc.as_dict(),
            event=method
        )

def send_webhook(webhook_name, doc_data, event):
    """Send webhook in background"""
    
    webhook = frappe.get_doc("Webhook", webhook_name)
    
    # Prepare payload
    if webhook.webhook_json:
        # Use Jinja template
        from frappe.utils.jinja import render_template
        payload = render_template(webhook.webhook_json, {"doc": doc_data})
        payload = frappe.parse_json(payload)
    else:
        # Default payload
        payload = {
            "event": event,
            "doctype": doc_data["doctype"],
            "name": doc_data["name"],
            "data": doc_data
        }
    
    # Prepare headers
    headers = {"Content-Type": "application/json"}
    
    # Add signature if secret is configured
    if webhook.webhook_secret:
        signature = generate_webhook_signature(
            frappe.as_json(payload), 
            webhook.webhook_secret
        )
        headers[WEBHOOK_SECRET_HEADER] = signature
    
    try:
        # Send webhook
        response = requests.post(
            webhook.request_url,
            json=payload,
            headers=headers,
            timeout=webhook.timeout or 30
        )
        
        response.raise_for_status()
        
        # Log success
        frappe.logger().info(f"Webhook sent successfully: {webhook_name}")
        
    except requests.exceptions.RequestException as e:
        # Log error and create integration request log
        frappe.log_error(
            message=f"Webhook failed: {str(e)}",
            title=f"Webhook Error: {webhook_name}"
        )
```

## Third-Party Integrations

### HTTP Client Utilities

Based on `frappe/integrations/utils.py:12-56`:

```python
def make_request(method, url, auth=None, headers=None, data=None, json=None, params=None):
    """Standardized HTTP request method"""
    
    try:
        s = get_request_session()  # Reuse session with connection pooling
        response = s.request(
            method, url, 
            data=data, 
            auth=auth, 
            headers=headers, 
            json=json, 
            params=params
        )
        response.raise_for_status()
        
        # Handle different content types
        content_type = response.headers.get("content-type", "")
        if content_type == "text/plain; charset=utf-8":
            return parse_qs(response.text)
        elif "json" in content_type:
            return response.json()
        elif response.text:
            return response.text
            
    except Exception as exc:
        frappe.log_error()
        raise exc
```

### Integration Request Logging

```python
def create_integration_log(data, integration_type=None, service_name=None, **kwargs):
    """Log integration requests for debugging"""
    
    log = frappe.get_doc({
        "doctype": "Integration Request",
        "integration_type": integration_type,
        "service_name": service_name,
        "data": frappe.as_json(data),
        "request_headers": frappe.as_json(kwargs.get("headers", {})),
        "output": frappe.as_json(kwargs.get("output")),
        "error": kwargs.get("error"),
        "status": "Completed" if not kwargs.get("error") else "Failed"
    })
    
    log.insert(ignore_permissions=True)
    return log
```

### Payment Gateway Integration Example

```python
# apps/library_management/library_management/integrations/payment_gateway.py
import frappe
import requests
import hashlib
from frappe import _

class PaymentGateway:
    def __init__(self, gateway_settings):
        self.base_url = gateway_settings.api_url
        self.api_key = gateway_settings.api_key
        self.api_secret = gateway_settings.api_secret
        
    def process_payment(self, amount, member, description):
        """Process payment for library fines"""
        
        # Prepare payment data
        payment_data = {
            "amount": amount,
            "currency": "USD",
            "customer_id": member,
            "description": description,
            "return_url": f"{frappe.utils.get_url()}/library/payment/success",
            "cancel_url": f"{frappe.utils.get_url()}/library/payment/cancel"
        }
        
        # Generate signature
        payment_data["signature"] = self._generate_signature(payment_data)
        
        try:
            # Make payment request
            response = requests.post(
                f"{self.base_url}/payments",
                json=payment_data,
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json"
                },
                timeout=30
            )
            
            response.raise_for_status()
            result = response.json()
            
            # Create payment record
            payment_doc = frappe.get_doc({
                "doctype": "Library Payment",
                "member": member,
                "amount": amount,
                "payment_gateway": "Stripe",
                "gateway_transaction_id": result.get("transaction_id"),
                "status": "Pending",
                "gateway_response": frappe.as_json(result)
            })
            
            payment_doc.insert()
            
            return {
                "success": True,
                "payment_id": payment_doc.name,
                "redirect_url": result.get("payment_url")
            }
            
        except requests.exceptions.RequestException as e:
            frappe.log_error(
                message=f"Payment gateway error: {str(e)}",
                title="Payment Processing Failed"
            )
            return {
                "success": False,
                "error": _("Payment processing failed. Please try again.")
            }
    
    def _generate_signature(self, data):
        """Generate payment signature for security"""
        import hmac
        
        # Sort data for consistent signature
        sorted_data = "&".join([f"{k}={v}" for k, v in sorted(data.items())])
        signature = hmac.new(
            self.api_secret.encode(),
            sorted_data.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return signature
```

## Data Import/Export Mechanisms

### Data Import System

Based on `frappe/core/doctype/data_import/importer.py` analysis:

```python
class DataImporter:
    def __init__(self, doctype, file_path, import_type="Insert"):
        self.doctype = doctype
        self.file_path = file_path
        self.import_type = import_type
        
    def import_data(self):
        """Import data from CSV/Excel file"""
        
        # Read file data
        data = self.read_file()
        
        # Validate data
        self.validate_data(data)
        
        # Process in batches
        batch_size = 100
        total_rows = len(data)
        success_count = 0
        error_count = 0
        errors = []
        
        for i in range(0, total_rows, batch_size):
            batch = data[i:i + batch_size]
            
            for row_idx, row in enumerate(batch):
                try:
                    # Create/update document
                    doc = self.process_row(row, i + row_idx + 1)
                    if doc:
                        success_count += 1
                        
                except Exception as e:
                    error_count += 1
                    errors.append({
                        "row": i + row_idx + 1,
                        "error": str(e),
                        "data": row
                    })
            
            # Commit batch
            frappe.db.commit()
            
            # Update progress
            frappe.publish_progress(
                (i + len(batch)) / total_rows * 100,
                title=f"Importing {self.doctype}",
                description=f"Processed {i + len(batch)} of {total_rows} rows"
            )
        
        return {
            "success": True,
            "total": total_rows,
            "success_count": success_count,
            "error_count": error_count,
            "errors": errors[:50]  # Limit error details
        }
```

### Data Export System

```python
@frappe.whitelist()
def export_library_data(doctype, filters=None, fields=None, file_format="Excel"):
    """Export library data to Excel/CSV"""
    
    # Permission check
    if not frappe.has_permission(doctype, "export"):
        frappe.throw(_("Not permitted to export data"))
    
    # Get data
    data = frappe.get_all(
        doctype,
        filters=filters or {},
        fields=fields or ["*"],
        limit_page_length=0  # No limit for export
    )
    
    if not data:
        frappe.throw(_("No data found to export"))
    
    # Create export file
    from frappe.utils.xlsxutils import make_xlsx
    from frappe.utils.csvutils import to_csv
    
    if file_format == "Excel":
        # Create Excel file
        xlsx_file = make_xlsx(
            data, 
            doctype,
            wb=None,
            column_widths=None
        )
        
        # Save file
        file_name = f"{doctype}_export_{frappe.utils.now()}.xlsx"
        file_doc = frappe.get_doc({
            "doctype": "File",
            "file_name": file_name,
            "content": xlsx_file.getvalue(),
            "is_private": 1
        })
        file_doc.insert()
        
    else:  # CSV format
        csv_content = to_csv(data)
        
        file_name = f"{doctype}_export_{frappe.utils.now()}.csv"
        file_doc = frappe.get_doc({
            "doctype": "File", 
            "file_name": file_name,
            "content": csv_content,
            "is_private": 1
        })
        file_doc.insert()
    
    return {
        "success": True,
        "file_url": file_doc.file_url,
        "file_name": file_doc.file_name,
        "records_count": len(data)
    }
```

## OAuth & Social Login

### OAuth 2.0 Provider Configuration

Based on integrations analysis:

```python
# OAuth Client Setup
{
    "doctype": "OAuth Client",
    "client_name": "Mobile App",
    "client_id": "generated_client_id",
    "client_secret": "generated_secret", 
    "grant_types": [
        "authorization_code",
        "refresh_token",
        "client_credentials"
    ],
    "redirect_uris": [
        "https://myapp.com/oauth/callback"
    ],
    "scopes": ["read", "write", "delete"],
    "skip_authorization": 0
}
```

### Social Login Integration

```python
# Google OAuth integration
def google_login_callback(code, state):
    """Handle Google OAuth callback"""
    
    try:
        # Exchange code for access token
        token_response = requests.post(
            "https://oauth2.googleapis.com/token",
            data={
                "client_id": google_settings.client_id,
                "client_secret": google_settings.client_secret,
                "code": code,
                "grant_type": "authorization_code",
                "redirect_uri": google_settings.redirect_uri
            }
        )
        
        token_data = token_response.json()
        access_token = token_data.get("access_token")
        
        # Get user info from Google
        user_response = requests.get(
            "https://www.googleapis.com/oauth2/v2/userinfo",
            headers={"Authorization": f"Bearer {access_token}"}
        )
        
        user_info = user_response.json()
        
        # Create or update user
        user = create_or_update_social_user(
            provider="google",
            email=user_info["email"],
            first_name=user_info["given_name"],
            last_name=user_info["family_name"],
            provider_id=user_info["id"]
        )
        
        # Login user
        frappe.local.login_manager.user = user
        frappe.local.login_manager.post_login()
        
        return {"success": True, "user": user}
        
    except Exception as e:
        frappe.log_error(f"Google login failed: {str(e)}")
        return {"success": False, "error": "Login failed"}
```

## Rate Limiting & Security

### Rate Limiting Implementation

```python
@frappe.whitelist()
@frappe.rate_limit(limit=60, window=60)  # 60 requests per minute
def search_api(query):
    """Rate-limited search API"""
    return perform_search(query)

# Custom rate limiting logic
def custom_rate_limit(key_func, limit, window):
    """Custom rate limiting decorator"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            # Generate rate limit key
            key = key_func(*args, **kwargs)
            cache_key = f"rate_limit:{key}"
            
            # Check current count
            current_count = frappe.cache.get(cache_key) or 0
            
            if current_count >= limit:
                frappe.throw(
                    _("Rate limit exceeded. Try again later."),
                    frappe.RateLimitExceededError
                )
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Increment counter
            frappe.cache.set(cache_key, current_count + 1, expires_in_sec=window)
            
            return result
        return wrapper
    return decorator

# Usage
@custom_rate_limit(
    key_func=lambda: frappe.session.user,
    limit=100,
    window=3600  # 1 hour
)
def expensive_api():
    return perform_expensive_operation()
```

### API Security Best Practices

```python
def secure_api_endpoint():
    """Security checklist for API endpoints"""
    
    # 1. Input validation
    def validate_input(data):
        # Sanitize HTML
        for key, value in data.items():
            if isinstance(value, str):
                data[key] = frappe.utils.sanitize_html(value)
        
        # Validate required fields
        required_fields = ["name", "email"]
        for field in required_fields:
            if not data.get(field):
                frappe.throw(_(f"{field} is required"))
        
        return data
    
    # 2. Permission validation
    def check_permissions(doctype, action):
        if not frappe.has_permission(doctype, action):
            frappe.throw(_("Insufficient permissions"), frappe.PermissionError)
    
    # 3. CSRF validation (automatic for unsafe methods)
    # 4. Rate limiting (via decorator)
    # 5. SQL injection prevention (use parameterized queries)
    # 6. XSS prevention (sanitize output)
    
    return {
        "validate_input": validate_input,
        "check_permissions": check_permissions
    }
```

## API Testing & Documentation

### API Testing Patterns

```python
# test_api.py
import frappe
import unittest
import json

class TestLibraryAPI(unittest.TestCase):
    
    def setUp(self):
        """Set up test data"""
        self.test_member = frappe.get_doc({
            "doctype": "Library Member",
            "member_name": "Test Member",
            "email": "test@example.com",
            "membership_type": "Standard"
        }).insert()
        
        # Create API client
        self.client = frappe.client.FrappeClient("http://localhost:8000")
        self.client.authenticate("test@example.com", "password")
    
    def test_get_available_books(self):
        """Test get available books API"""
        response = self.client.get_api("library_management.api.get_available_books")
        
        self.assertTrue(response.get("success"))
        self.assertIsInstance(response.get("books"), list)
    
    def test_issue_book_api(self):
        """Test book issue API"""
        # Create test book
        book = frappe.get_doc({
            "doctype": "Library Book",
            "title": "Test Book",
            "author": "Test Author",
            "status": "Available"
        }).insert()
        
        # Test API call
        response = self.client.post_api(
            "library_management.api.issue_book_to_member",
            {
                "book": book.name,
                "member": self.test_member.name
            }
        )
        
        self.assertTrue(response.get("success"))
        self.assertIsNotNone(response.get("issue_id"))
    
    def test_api_rate_limiting(self):
        """Test rate limiting functionality"""
        # Make requests up to limit
        for i in range(60):
            response = self.client.get_api("library_management.api.search_books", {"query": "test"})
            self.assertTrue(response.get("success"))
        
        # 61st request should fail
        with self.assertRaises(frappe.RateLimitExceededError):
            self.client.get_api("library_management.api.search_books", {"query": "test"})
    
    def tearDown(self):
        """Clean up test data"""
        frappe.db.delete("Library Member", {"email": "test@example.com"})
        frappe.db.delete("Library Book", {"title": "Test Book"})
```

### API Documentation Generation

```python
def generate_api_docs():
    """Generate API documentation from whitelisted methods"""
    
    import inspect
    from frappe.utils.autodoc import get_whitelisted_methods
    
    api_docs = {
        "endpoints": [],
        "version": "1.0",
        "base_url": frappe.utils.get_url()
    }
    
    # Get all whitelisted methods
    methods = get_whitelisted_methods()
    
    for module_path, method_name in methods:
        try:
            # Get method object
            method = frappe.get_attr(f"{module_path}.{method_name}")
            
            # Extract documentation
            doc = {
                "path": f"/api/method/{module_path}.{method_name}",
                "method": "GET" if getattr(method, "safe", False) else "POST",
                "description": inspect.getdoc(method),
                "parameters": get_method_parameters(method)
            }
            
            api_docs["endpoints"].append(doc)
            
        except Exception:
            continue
    
    return api_docs

def get_method_parameters(method):
    """Extract method parameters for documentation"""
    import inspect
    
    sig = inspect.signature(method)
    params = []
    
    for name, param in sig.parameters.items():
        param_doc = {
            "name": name,
            "required": param.default == inspect.Parameter.empty,
            "type": str(param.annotation) if param.annotation != inspect.Parameter.empty else "any"
        }
        params.append(param_doc)
    
    return params
```

---

**Next Steps**: Continue with [Permissions & Security](06-permissions-and-security.md) to learn about role-based access control, user management, and security implementation patterns.