# Security Implementation Guide - ERPNext Enterprise Security Patterns

*Based on ERPNext security analysis - Comprehensive guide for implementing enterprise-grade security in Frappe framework applications*

## 🎯 Overview

This guide provides battle-tested security implementation patterns extracted from ERPNext's production security architecture. These patterns ensure your Frappe applications maintain enterprise-grade security standards, protecting against common vulnerabilities while maintaining usability and performance.

---

## 📋 Table of Contents

1. [Authentication and Authorization](#authentication-and-authorization)
2. [Input Validation and Sanitization](#input-validation-and-sanitization)
3. [Data Security and Encryption](#data-security-and-encryption)
4. [API Security Implementation](#api-security-implementation)
5. [Session Security Management](#session-security-management)
6. [Database Security Patterns](#database-security-patterns)
7. [File Upload Security](#file-upload-security)
8. [Audit Logging and Monitoring](#audit-logging-and-monitoring)
9. [Security Testing Methodologies](#security-testing-methodologies)
10. [Compliance and Regulatory Security](#compliance-and-regulatory-security)
11. [Security Incident Response](#security-incident-response)

---

## 🔐 Authentication and Authorization

### Role-Based Access Control (RBAC) Implementation
*Based on ERPNext's permission system*

```python
# rbac_implementation.py
import frappe
from frappe import _

class SecurityManager:
    """
    Enterprise-grade security management
    Based on ERPNext's role-based access control patterns
    """
    
    def __init__(self):
        self.permission_cache = {}
        self.role_hierarchy = self.load_role_hierarchy()
    
    def has_permission(self, doctype, ptype, doc=None, user=None):
        """
        Enhanced permission checking with caching and logging
        Pattern from ERPNext's permission system
        """
        
        if not user:
            user = frappe.session.user
        
        # System Manager has all permissions
        if "System Manager" in frappe.get_roles(user):
            return True
        
        # Check permission cache
        cache_key = f"{user}:{doctype}:{ptype}:{doc.name if doc else 'None'}"
        
        if cache_key in self.permission_cache:
            return self.permission_cache[cache_key]
        
        # Execute permission check
        permission_result = self._check_permission(doctype, ptype, doc, user)
        
        # Cache result
        self.permission_cache[cache_key] = permission_result
        
        # Log permission check for audit
        self.log_permission_check(user, doctype, ptype, doc, permission_result)
        
        return permission_result
    
    def _check_permission(self, doctype, ptype, doc, user):
        """Core permission checking logic"""
        
        # Get user roles
        user_roles = frappe.get_roles(user)
        
        # Check DocType-level permissions
        doctype_permissions = frappe.get_all(
            "DocPerm",
            filters={
                "parent": doctype,
                "role": ["in", user_roles],
                ptype: 1
            }
        )
        
        if not doctype_permissions:
            return False
        
        # Document-level permission checks
        if doc:
            return self._check_document_permissions(doc, user, user_roles, doctype_permissions)
        
        return True
    
    def _check_document_permissions(self, doc, user, user_roles, doctype_permissions):
        """Check document-level permissions"""
        
        for perm in doctype_permissions:
            # Check if conditions match
            if self._check_permission_conditions(doc, user, perm):
                return True
        
        return False
    
    def _check_permission_conditions(self, doc, user, perm):
        """Check specific permission conditions"""
        
        # Owner check
        if perm.get("if_owner") and doc.owner != user:
            return False
        
        # User-specific permission
        if perm.get("user_permission"):
            if not self._has_user_permission(user, doc):
                return False
        
        # Company-specific permission
        if hasattr(doc, "company"):
            user_companies = self.get_user_companies(user)
            if user_companies and doc.company not in user_companies:
                return False
        
        # Territory/Department restrictions
        if hasattr(doc, "territory"):
            user_territories = self.get_user_territories(user)
            if user_territories and doc.territory not in user_territories:
                return False
        
        return True
    
    def validate_api_permissions(self, method_name, args=None):
        """
        Validate API method permissions
        Pattern from ERPNext's @frappe.whitelist() usage
        """
        
        # Check if method requires authentication
        if not frappe.session.user or frappe.session.user == "Guest":
            if not self._is_public_method(method_name):
                frappe.throw(_("Authentication required"), frappe.AuthenticationError)
        
        # Check method-specific permissions
        required_permissions = self._get_method_permissions(method_name)
        
        for perm_check in required_permissions:
            if not self.has_permission(
                perm_check["doctype"], 
                perm_check["ptype"],
                args.get("doc") if args else None
            ):
                frappe.throw(
                    _("Insufficient permissions for {0}").format(method_name),
                    frappe.PermissionError
                )
    
    def _has_user_permission(self, user, doc):
        """Check user-specific permissions"""
        
        # Check User Permission records
        user_permissions = frappe.get_all(
            "User Permission",
            filters={
                "user": user,
                "allow": doc.doctype,
                "for_value": doc.name
            }
        )
        
        return len(user_permissions) > 0
    
    def get_user_companies(self, user):
        """Get companies user has access to"""
        
        user_permissions = frappe.get_all(
            "User Permission",
            filters={
                "user": user,
                "allow": "Company"
            },
            fields=["for_value"]
        )
        
        return [perm.for_value for perm in user_permissions]
    
    def get_user_territories(self, user):
        """Get territories user has access to"""
        
        user_permissions = frappe.get_all(
            "User Permission",
            filters={
                "user": user,
                "allow": "Territory"
            },
            fields=["for_value"]
        )
        
        return [perm.for_value for perm in user_permissions]
    
    def log_permission_check(self, user, doctype, ptype, doc, result):
        """Log permission check for audit trail"""
        
        log_entry = {
            "doctype": "Permission Check Log",
            "user": user,
            "document_type": doctype,
            "permission_type": ptype,
            "document_name": doc.name if doc else None,
            "result": "Allowed" if result else "Denied",
            "timestamp": frappe.utils.now(),
            "ip_address": frappe.local.request.headers.get("X-Real-IP") or frappe.local.request.remote_addr,
            "user_agent": frappe.local.request.headers.get("User-Agent", "")
        }
        
        # Store in database for audit
        frappe.get_doc(log_entry).insert(ignore_permissions=True)
    
    def load_role_hierarchy(self):
        """Load role hierarchy for inheritance"""
        
        return {
            "System Manager": ["Administrator"],
            "Sales Manager": ["Sales User", "Employee"],
            "Purchase Manager": ["Purchase User", "Employee"],
            "Accounts Manager": ["Accounts User", "Employee"],
            "HR Manager": ["HR User", "Employee"],
            "Stock Manager": ["Stock User", "Employee"]
        }

# Document-level Security Implementation
class SecureDocument(frappe.model.document.Document):
    """
    Base class for secure document implementation
    Pattern from ERPNext's controller validations
    """
    
    def validate(self):
        """Enhanced validation with security checks"""
        super().validate()
        
        # Validate user permissions
        self.validate_user_permissions()
        
        # Validate data integrity
        self.validate_data_integrity()
        
        # Validate business rules
        self.validate_business_rules()
        
        # Check for suspicious activities
        self.check_suspicious_activities()
    
    def validate_user_permissions(self):
        """Validate user has permission to modify document"""
        
        security_manager = SecurityManager()
        
        # Check if user can create/update this document type
        if self.is_new():
            if not security_manager.has_permission(self.doctype, "create"):
                frappe.throw(_("No permission to create {0}").format(self.doctype))
        else:
            if not security_manager.has_permission(self.doctype, "write", self):
                frappe.throw(_("No permission to modify {0}").format(self.name))
    
    def validate_data_integrity(self):
        """Validate data integrity and prevent tampering"""
        
        # Check for required field tampering
        required_fields = self.get_required_fields()
        for field in required_fields:
            if not self.get(field):
                frappe.throw(_("{0} is required").format(self.meta.get_label(field)))
        
        # Validate field value ranges
        self.validate_numeric_ranges()
        
        # Validate reference data integrity
        self.validate_reference_integrity()
    
    def validate_business_rules(self):
        """Validate business-specific security rules"""
        
        # Company-specific validation
        if hasattr(self, "company"):
            user_companies = SecurityManager().get_user_companies(frappe.session.user)
            if user_companies and self.company not in user_companies:
                frappe.throw(_("Access denied for company {0}").format(self.company))
        
        # Date-based restrictions
        if hasattr(self, "posting_date"):
            self.validate_posting_date_permissions()
        
        # Amount-based restrictions
        if hasattr(self, "grand_total") or hasattr(self, "amount"):
            self.validate_amount_permissions()
    
    def check_suspicious_activities(self):
        """Check for suspicious modification patterns"""
        
        if not self.is_new():
            # Check for rapid modifications
            if self.modified and self.creation:
                time_diff = frappe.utils.time_diff_in_seconds(
                    frappe.utils.now(), self.creation
                )
                
                if time_diff < 60:  # Modified within 1 minute of creation
                    self.log_suspicious_activity("Rapid modification after creation")
            
            # Check for bulk field changes
            changed_fields = self.get_doc_before_save() and len([
                field for field in self.meta.fields
                if self.get(field.fieldname) != self.get_doc_before_save().get(field.fieldname)
            ]) or 0
            
            if changed_fields > 10:  # More than 10 fields changed
                self.log_suspicious_activity(f"Bulk field modification: {changed_fields} fields")
    
    def validate_numeric_ranges(self):
        """Validate numeric field ranges"""
        
        numeric_validations = {
            "rate": {"min": 0, "max": 999999999},
            "qty": {"min": 0, "max": 999999999},
            "amount": {"min": 0, "max": 999999999},
            "discount_percentage": {"min": 0, "max": 100}
        }
        
        for field, validation in numeric_validations.items():
            if hasattr(self, field) and self.get(field) is not None:
                value = self.get(field)
                if value < validation["min"] or value > validation["max"]:
                    frappe.throw(
                        _("{0} must be between {1} and {2}").format(
                            self.meta.get_label(field),
                            validation["min"],
                            validation["max"]
                        )
                    )
    
    def validate_reference_integrity(self):
        """Validate reference field integrity"""
        
        for field in self.meta.fields:
            if field.fieldtype == "Link" and self.get(field.fieldname):
                # Validate link field exists and is accessible
                if not frappe.db.exists(field.options, self.get(field.fieldname)):
                    frappe.throw(
                        _("Invalid {0}: {1}").format(
                            field.label, self.get(field.fieldname)
                        )
                    )
                
                # Check if user has permission to reference this document
                if not SecurityManager().has_permission(field.options, "read"):
                    frappe.throw(
                        _("No permission to reference {0}").format(field.options)
                    )
    
    def validate_posting_date_permissions(self):
        """Validate posting date permissions"""
        
        user_roles = frappe.get_roles(frappe.session.user)
        
        # Check if user can set backdated entries
        if "Accounts Manager" not in user_roles:
            max_days_back = frappe.db.get_single_value("System Settings", "max_days_backdated_posting") or 5
            
            if self.posting_date < frappe.utils.add_days(frappe.utils.nowdate(), -max_days_back):
                frappe.throw(
                    _("Posting date cannot be more than {0} days back").format(max_days_back)
                )
    
    def validate_amount_permissions(self):
        """Validate amount-based permissions"""
        
        amount = self.get("grand_total") or self.get("amount") or 0
        user = frappe.session.user
        
        # Check user's authorization limit
        user_limit = frappe.db.get_value("User", user, "authorization_limit") or 0
        
        if amount > user_limit:
            # Check if there's an approver available
            approvers = frappe.get_all(
                "User",
                filters={
                    "authorization_limit": [">=", amount],
                    "enabled": 1
                },
                limit=1
            )
            
            if not approvers:
                frappe.throw(
                    _("Amount {0} exceeds authorization limit. No approver available.").format(
                        frappe.format_value(amount, "Currency")
                    )
                )
    
    def log_suspicious_activity(self, activity_type):
        """Log suspicious activity for security monitoring"""
        
        log_entry = {
            "doctype": "Security Alert",
            "alert_type": "Suspicious Activity",
            "document_type": self.doctype,
            "document_name": self.name,
            "user": frappe.session.user,
            "activity_description": activity_type,
            "timestamp": frappe.utils.now(),
            "ip_address": frappe.local.request.headers.get("X-Real-IP") or frappe.local.request.remote_addr,
            "severity": "Medium"
        }
        
        frappe.get_doc(log_entry).insert(ignore_permissions=True)

# Multi-Factor Authentication Implementation
class MFAManager:
    """Multi-Factor Authentication implementation"""
    
    def __init__(self):
        self.otp_validity = 300  # 5 minutes
        
    def enable_mfa_for_user(self, user, method="totp"):
        """Enable MFA for user"""
        
        if method == "totp":
            return self.setup_totp(user)
        elif method == "sms":
            return self.setup_sms_mfa(user)
        elif method == "email":
            return self.setup_email_mfa(user)
    
    def setup_totp(self, user):
        """Setup TOTP (Time-based OTP) for user"""
        
        import pyotp
        import qrcode
        import io
        import base64
        
        # Generate secret key
        secret = pyotp.random_base32()
        
        # Create TOTP object
        totp = pyotp.TOTP(secret)
        
        # Generate QR code
        provisioning_uri = totp.provisioning_uri(
            name=user,
            issuer_name="Your App Name"
        )
        
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(provisioning_uri)
        qr.make(fit=True)
        
        img = qr.make_image(fill_color="black", back_color="white")
        
        # Convert to base64 for display
        buffer = io.BytesIO()
        img.save(buffer, format='PNG')
        qr_code_data = base64.b64encode(buffer.getvalue()).decode()
        
        # Save MFA settings
        mfa_doc = frappe.get_doc({
            "doctype": "MFA Settings",
            "user": user,
            "method": "totp",
            "secret_key": secret,
            "enabled": 0,  # Enable after verification
            "backup_codes": self.generate_backup_codes()
        })
        mfa_doc.insert(ignore_permissions=True)
        
        return {
            "secret": secret,
            "qr_code": qr_code_data,
            "backup_codes": mfa_doc.backup_codes
        }
    
    def verify_mfa_token(self, user, token, method="totp"):
        """Verify MFA token"""
        
        mfa_settings = frappe.get_value(
            "MFA Settings",
            {"user": user, "method": method, "enabled": 1},
            ["secret_key", "backup_codes"],
            as_dict=True
        )
        
        if not mfa_settings:
            return False
        
        if method == "totp":
            import pyotp
            totp = pyotp.TOTP(mfa_settings.secret_key)
            
            # Verify current token
            if totp.verify(token):
                return True
            
            # Check backup codes
            backup_codes = frappe.parse_json(mfa_settings.backup_codes)
            if token in backup_codes:
                # Remove used backup code
                backup_codes.remove(token)
                frappe.db.set_value(
                    "MFA Settings", 
                    {"user": user, "method": method},
                    "backup_codes",
                    frappe.as_json(backup_codes)
                )
                return True
        
        return False
    
    def generate_backup_codes(self, count=10):
        """Generate backup codes for MFA"""
        
        import secrets
        import string
        
        backup_codes = []
        for _ in range(count):
            code = ''.join(secrets.choice(string.ascii_uppercase + string.digits) for _ in range(8))
            backup_codes.append(code)
        
        return backup_codes

# Password Security Implementation
class PasswordSecurityManager:
    """Password security management"""
    
    def __init__(self):
        self.min_length = 8
        self.require_uppercase = True
        self.require_lowercase = True
        self.require_numbers = True
        self.require_special = True
        self.password_history_count = 5
    
    def validate_password_strength(self, password):
        """Validate password meets security requirements"""
        
        errors = []
        
        if len(password) < self.min_length:
            errors.append(f"Password must be at least {self.min_length} characters long")
        
        if self.require_uppercase and not any(c.isupper() for c in password):
            errors.append("Password must contain at least one uppercase letter")
        
        if self.require_lowercase and not any(c.islower() for c in password):
            errors.append("Password must contain at least one lowercase letter")
        
        if self.require_numbers and not any(c.isdigit() for c in password):
            errors.append("Password must contain at least one number")
        
        if self.require_special and not any(c in "!@#$%^&*()_+-=[]{}|;:,.<>?" for c in password):
            errors.append("Password must contain at least one special character")
        
        # Check against common passwords
        if self.is_common_password(password):
            errors.append("Password is too common. Please choose a stronger password")
        
        return errors
    
    def check_password_history(self, user, new_password):
        """Check if password was used recently"""
        
        import bcrypt
        
        password_history = frappe.get_all(
            "Password History",
            filters={"user": user},
            fields=["password_hash"],
            order_by="creation desc",
            limit=self.password_history_count
        )
        
        for history_entry in password_history:
            if bcrypt.checkpw(new_password.encode('utf-8'), history_entry.password_hash.encode('utf-8')):
                return False
        
        return True
    
    def store_password_history(self, user, password_hash):
        """Store password in history"""
        
        # Create password history entry
        frappe.get_doc({
            "doctype": "Password History",
            "user": user,
            "password_hash": password_hash,
            "created_on": frappe.utils.now()
        }).insert(ignore_permissions=True)
        
        # Clean up old entries
        old_entries = frappe.get_all(
            "Password History",
            filters={"user": user},
            order_by="creation desc",
            limit_start=self.password_history_count
        )
        
        for entry in old_entries:
            frappe.delete_doc("Password History", entry.name, ignore_permissions=True)
    
    def is_common_password(self, password):
        """Check if password is in common password list"""
        
        common_passwords = [
            "password", "123456", "password123", "admin", "qwerty",
            "letmein", "welcome", "monkey", "dragon", "master"
        ]
        
        return password.lower() in common_passwords
    
    def generate_secure_password(self, length=12):
        """Generate a secure password"""
        
        import secrets
        import string
        
        # Ensure at least one character from each required set
        password = []
        
        if self.require_uppercase:
            password.append(secrets.choice(string.ascii_uppercase))
        
        if self.require_lowercase:
            password.append(secrets.choice(string.ascii_lowercase))
        
        if self.require_numbers:
            password.append(secrets.choice(string.digits))
        
        if self.require_special:
            password.append(secrets.choice("!@#$%^&*()_+-=[]{}|;:,.<>?"))
        
        # Fill remaining length with random characters
        all_chars = string.ascii_letters + string.digits + "!@#$%^&*()_+-=[]{}|;:,.<>?"
        for _ in range(length - len(password)):
            password.append(secrets.choice(all_chars))
        
        # Shuffle the password
        secrets.SystemRandom().shuffle(password)
        
        return ''.join(password)
```

---

## 🛡️ Input Validation and Sanitization

### Comprehensive Input Validation Framework
*Based on ERPNext's validation patterns*

```python
# input_validation.py
import frappe
import re
import html
from frappe import _

class InputValidator:
    """
    Comprehensive input validation and sanitization
    Pattern from ERPNext's validate methods
    """
    
    def __init__(self):
        self.validation_rules = self.load_validation_rules()
        self.sanitization_rules = self.load_sanitization_rules()
    
    def validate_and_sanitize(self, data, validation_schema):
        """Validate and sanitize input data"""
        
        errors = []
        sanitized_data = {}
        
        for field, schema in validation_schema.items():
            if field in data:
                value = data[field]
                
                # Sanitize input
                sanitized_value = self.sanitize_input(value, schema.get("type", "string"))
                
                # Validate input
                field_errors = self.validate_field(sanitized_value, schema, field)
                
                if field_errors:
                    errors.extend(field_errors)
                else:
                    sanitized_data[field] = sanitized_value
            
            elif schema.get("required", False):
                errors.append(f"{field} is required")
        
        return {
            "valid": len(errors) == 0,
            "errors": errors,
            "data": sanitized_data
        }
    
    def sanitize_input(self, value, input_type):
        """Sanitize input based on type"""
        
        if value is None:
            return None
        
        if input_type == "string":
            return self.sanitize_string(value)
        elif input_type == "html":
            return self.sanitize_html(value)
        elif input_type == "email":
            return self.sanitize_email(value)
        elif input_type == "phone":
            return self.sanitize_phone(value)
        elif input_type == "numeric":
            return self.sanitize_numeric(value)
        elif input_type == "alphanumeric":
            return self.sanitize_alphanumeric(value)
        
        return str(value)
    
    def sanitize_string(self, value):
        """Sanitize string input"""
        
        if not isinstance(value, str):
            value = str(value)
        
        # Remove null bytes
        value = value.replace('\x00', '')
        
        # Escape HTML entities
        value = html.escape(value)
        
        # Remove excessive whitespace
        value = re.sub(r'\s+', ' ', value.strip())
        
        return value
    
    def sanitize_html(self, value):
        """Sanitize HTML input using allowlist"""
        
        try:
            import bleach
            
            allowed_tags = [
                'p', 'br', 'strong', 'em', 'u', 'ol', 'ul', 'li',
                'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'blockquote',
                'a', 'img', 'table', 'thead', 'tbody', 'tr', 'td', 'th'
            ]
            
            allowed_attributes = {
                'a': ['href', 'title'],
                'img': ['src', 'alt', 'width', 'height'],
                'table': ['class'],
                '*': ['class', 'style']
            }
            
            return bleach.clean(
                value,
                tags=allowed_tags,
                attributes=allowed_attributes,
                strip=True
            )
            
        except ImportError:
            # Fallback to basic HTML escaping
            return html.escape(value)
    
    def sanitize_email(self, value):
        """Sanitize email input"""
        
        if not isinstance(value, str):
            value = str(value)
        
        # Convert to lowercase
        value = value.lower().strip()
        
        # Remove dangerous characters
        value = re.sub(r'[<>"\'\s]', '', value)
        
        return value
    
    def sanitize_phone(self, value):
        """Sanitize phone number input"""
        
        if not isinstance(value, str):
            value = str(value)
        
        # Keep only digits, +, -, (, ), and spaces
        value = re.sub(r'[^\d+\-() ]', '', value)
        
        return value.strip()
    
    def sanitize_numeric(self, value):
        """Sanitize numeric input"""
        
        if isinstance(value, (int, float)):
            return value
        
        if isinstance(value, str):
            # Remove non-numeric characters except decimal point and minus
            value = re.sub(r'[^\d.-]', '', value)
            
            try:
                if '.' in value:
                    return float(value)
                else:
                    return int(value)
            except ValueError:
                return 0
        
        return 0
    
    def sanitize_alphanumeric(self, value):
        """Sanitize to alphanumeric only"""
        
        if not isinstance(value, str):
            value = str(value)
        
        # Keep only alphanumeric characters
        value = re.sub(r'[^a-zA-Z0-9]', '', value)
        
        return value
    
    def validate_field(self, value, schema, field_name):
        """Validate individual field"""
        
        errors = []
        
        # Required validation
        if schema.get("required", False) and not value:
            errors.append(f"{field_name} is required")
            return errors
        
        if value is None or value == "":
            return errors
        
        # Type-specific validations
        field_type = schema.get("type", "string")
        
        if field_type == "email":
            if not self.is_valid_email(value):
                errors.append(f"{field_name} must be a valid email address")
        
        elif field_type == "phone":
            if not self.is_valid_phone(value):
                errors.append(f"{field_name} must be a valid phone number")
        
        elif field_type == "numeric":
            if not isinstance(value, (int, float)):
                errors.append(f"{field_name} must be a number")
            else:
                # Range validation
                if "min" in schema and value < schema["min"]:
                    errors.append(f"{field_name} must be at least {schema['min']}")
                
                if "max" in schema and value > schema["max"]:
                    errors.append(f"{field_name} must be at most {schema['max']}")
        
        elif field_type == "string":
            # Length validation
            if "min_length" in schema and len(value) < schema["min_length"]:
                errors.append(f"{field_name} must be at least {schema['min_length']} characters")
            
            if "max_length" in schema and len(value) > schema["max_length"]:
                errors.append(f"{field_name} must be at most {schema['max_length']} characters")
            
            # Pattern validation
            if "pattern" in schema:
                if not re.match(schema["pattern"], value):
                    errors.append(f"{field_name} format is invalid")
        
        # Custom validation function
        if "validator" in schema:
            custom_error = schema["validator"](value)
            if custom_error:
                errors.append(custom_error)
        
        return errors
    
    def is_valid_email(self, email):
        """Validate email format"""
        
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(email_pattern, email) is not None
    
    def is_valid_phone(self, phone):
        """Validate phone number format"""
        
        # Basic phone validation - can be enhanced based on requirements
        phone_pattern = r'^\+?[\d\s\-\(\)]{7,15}$'
        return re.match(phone_pattern, phone) is not None
    
    def load_validation_rules(self):
        """Load validation rules from configuration"""
        
        return {
            "customer_name": {
                "type": "string",
                "required": True,
                "min_length": 2,
                "max_length": 100,
                "pattern": r'^[a-zA-Z\s\-\.]+$'
            },
            "email": {
                "type": "email",
                "required": True
            },
            "phone": {
                "type": "phone",
                "required": False
            },
            "amount": {
                "type": "numeric",
                "required": True,
                "min": 0,
                "max": 999999999
            }
        }
    
    def load_sanitization_rules(self):
        """Load sanitization rules"""
        
        return {
            "remove_scripts": True,
            "escape_html": True,
            "trim_whitespace": True,
            "remove_null_bytes": True
        }

# SQL Injection Prevention
class SQLInjectionPrevention:
    """Prevent SQL injection attacks"""
    
    @staticmethod
    def validate_query_params(params):
        """Validate parameters for SQL queries"""
        
        dangerous_patterns = [
            r"['\";]",  # Quote characters
            r"--",      # SQL comments
            r"/\*.*\*/", # Multi-line comments
            r"\b(union|select|insert|update|delete|drop|create|alter|exec|execute)\b",  # SQL keywords
            r"<script", # Script tags
            r"javascript:", # JavaScript protocol
        ]
        
        for param in params:
            if isinstance(param, str):
                for pattern in dangerous_patterns:
                    if re.search(pattern, param, re.IGNORECASE):
                        frappe.throw(
                            _("Invalid characters detected in input"),
                            frappe.SecurityValidationError
                        )
    
    @staticmethod
    def safe_query(query, params=None):
        """Execute parameterized query safely"""
        
        if params:
            SQLInjectionPrevention.validate_query_params(params)
            return frappe.db.sql(query, params, as_dict=True)
        else:
            return frappe.db.sql(query, as_dict=True)

# Cross-Site Scripting (XSS) Prevention
class XSSPrevention:
    """Prevent XSS attacks"""
    
    @staticmethod
    def sanitize_output(content, context="html"):
        """Sanitize content for output"""
        
        if context == "html":
            return html.escape(str(content))
        elif context == "js":
            return XSSPrevention.escape_js_string(str(content))
        elif context == "url":
            return XSSPrevention.escape_url(str(content))
        
        return str(content)
    
    @staticmethod
    def escape_js_string(value):
        """Escape string for JavaScript context"""
        
        # Escape special characters for JavaScript
        escape_map = {
            '\\': '\\\\',
            '"': '\\"',
            "'": "\\'",
            '\n': '\\n',
            '\r': '\\r',
            '\t': '\\t',
            '\f': '\\f',
            '\b': '\\b',
            '\v': '\\v',
            '\0': '\\0'
        }
        
        for char, escaped in escape_map.items():
            value = value.replace(char, escaped)
        
        return value
    
    @staticmethod
    def escape_url(value):
        """Escape value for URL context"""
        
        try:
            from urllib.parse import quote
            return quote(value, safe='')
        except ImportError:
            import urllib
            return urllib.quote(value, safe='')

# Content Security Policy Implementation
class CSPManager:
    """Content Security Policy management"""
    
    def __init__(self):
        self.default_policy = {
            "default-src": ["'self'"],
            "script-src": ["'self'", "'unsafe-inline'", "https://cdn.jsdelivr.net"],
            "style-src": ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
            "img-src": ["'self'", "data:", "https:"],
            "font-src": ["'self'", "https://fonts.gstatic.com"],
            "connect-src": ["'self'"],
            "frame-ancestors": ["'none'"],
            "base-uri": ["'self'"],
            "form-action": ["'self'"]
        }
    
    def generate_csp_header(self, custom_policy=None):
        """Generate CSP header"""
        
        policy = self.default_policy.copy()
        
        if custom_policy:
            for directive, sources in custom_policy.items():
                if directive in policy:
                    policy[directive].extend(sources)
                else:
                    policy[directive] = sources
        
        # Build CSP string
        csp_parts = []
        for directive, sources in policy.items():
            csp_parts.append(f"{directive} {' '.join(sources)}")
        
        return "; ".join(csp_parts)
    
    def set_csp_headers(self, custom_policy=None):
        """Set CSP headers in response"""
        
        csp_header = self.generate_csp_header(custom_policy)
        
        frappe.local.response.headers.update({
            "Content-Security-Policy": csp_header,
            "X-Content-Type-Options": "nosniff",
            "X-Frame-Options": "DENY",
            "X-XSS-Protection": "1; mode=block",
            "Referrer-Policy": "strict-origin-when-cross-origin"
        })

# CSRF Protection Implementation
class CSRFProtection:
    """CSRF attack protection"""
    
    @staticmethod
    def generate_csrf_token():
        """Generate CSRF token"""
        
        import secrets
        return secrets.token_urlsafe(32)
    
    @staticmethod
    def validate_csrf_token(token):
        """Validate CSRF token"""
        
        # Get token from session
        session_token = frappe.session.get("csrf_token")
        
        if not session_token:
            return False
        
        # Constant-time comparison
        return secrets.compare_digest(token, session_token)
    
    @staticmethod
    def require_csrf_token():
        """Decorator to require CSRF token validation"""
        
        def decorator(func):
            def wrapper(*args, **kwargs):
                # Skip CSRF check for GET requests
                if frappe.local.request.method == "GET":
                    return func(*args, **kwargs)
                
                # Check CSRF token
                csrf_token = frappe.form_dict.get("csrf_token") or \
                           frappe.local.request.headers.get("X-CSRFToken")
                
                if not csrf_token or not CSRFProtection.validate_csrf_token(csrf_token):
                    frappe.throw(
                        _("CSRF token validation failed"),
                        frappe.CSRFTokenError
                    )
                
                return func(*args, **kwargs)
            
            return wrapper
        return decorator

# File Upload Validation
class FileUploadValidator:
    """Validate file uploads for security"""
    
    def __init__(self):
        self.allowed_extensions = {
            'image': ['.jpg', '.jpeg', '.png', '.gif', '.bmp', '.webp'],
            'document': ['.pdf', '.doc', '.docx', '.xls', '.xlsx', '.ppt', '.pptx'],
            'text': ['.txt', '.csv', '.json', '.xml'],
            'archive': ['.zip', '.tar', '.gz']
        }
        
        self.max_file_size = 10 * 1024 * 1024  # 10MB
        
        self.dangerous_extensions = [
            '.exe', '.bat', '.com', '.cmd', '.scr', '.pif',
            '.js', '.vbs', '.jar', '.sh', '.py', '.php'
        ]
    
    def validate_file_upload(self, file_path, file_type=None):
        """Validate uploaded file"""
        
        import os
        import magic
        
        errors = []
        
        if not os.path.exists(file_path):
            errors.append("File does not exist")
            return errors
        
        # Check file size
        file_size = os.path.getsize(file_path)
        if file_size > self.max_file_size:
            errors.append(f"File size exceeds maximum limit of {self.max_file_size/1024/1024:.1f}MB")
        
        # Check file extension
        file_ext = os.path.splitext(file_path)[1].lower()
        
        if file_ext in self.dangerous_extensions:
            errors.append("File type not allowed for security reasons")
            return errors
        
        # Validate file type if specified
        if file_type:
            allowed_exts = self.allowed_extensions.get(file_type, [])
            if file_ext not in allowed_exts:
                errors.append(f"File type {file_ext} not allowed for {file_type} uploads")
        
        # Validate file content matches extension
        try:
            mime_type = magic.from_file(file_path, mime=True)
            if not self.is_mime_type_safe(mime_type, file_ext):
                errors.append("File content does not match file extension")
        except:
            # Magic library not available, skip MIME validation
            pass
        
        # Check for embedded executables
        if self.has_embedded_executable(file_path):
            errors.append("File contains suspicious content")
        
        return errors
    
    def is_mime_type_safe(self, mime_type, file_ext):
        """Check if MIME type matches file extension"""
        
        mime_mappings = {
            '.jpg': ['image/jpeg'],
            '.jpeg': ['image/jpeg'],
            '.png': ['image/png'],
            '.gif': ['image/gif'],
            '.pdf': ['application/pdf'],
            '.txt': ['text/plain'],
            '.csv': ['text/csv', 'application/csv']
        }
        
        allowed_mimes = mime_mappings.get(file_ext, [])
        
        return mime_type in allowed_mimes if allowed_mimes else True
    
    def has_embedded_executable(self, file_path):
        """Check for embedded executable content"""
        
        try:
            with open(file_path, 'rb') as f:
                # Read first 1KB to check for executable signatures
                content = f.read(1024)
                
                # Check for common executable signatures
                executable_signatures = [
                    b'\x4d\x5a',  # PE/COFF
                    b'\x7f\x45\x4c\x46',  # ELF
                    b'\xfe\xed\xfa\xce',  # Mach-O
                    b'<script',  # Script tags
                    b'javascript:',  # JavaScript URLs
                ]
                
                for signature in executable_signatures:
                    if signature in content.lower():
                        return True
            
            return False
            
        except Exception:
            return False

# API Input Validation Decorator
def validate_api_input(validation_schema):
    """Decorator for API input validation"""
    
    def decorator(func):
        def wrapper(*args, **kwargs):
            validator = InputValidator()
            
            # Get request data
            if frappe.local.request.method == "POST":
                data = frappe.local.form_dict
            else:
                data = frappe.local.request.args
            
            # Validate input
            result = validator.validate_and_sanitize(data, validation_schema)
            
            if not result["valid"]:
                frappe.throw(
                    _("Validation errors: {0}").format(", ".join(result["errors"])),
                    frappe.ValidationError
                )
            
            # Replace form_dict with sanitized data
            frappe.local.form_dict.update(result["data"])
            
            return func(*args, **kwargs)
        
        return wrapper
    return decorator

# Usage Examples
@frappe.whitelist()
@validate_api_input({
    "customer_name": {
        "type": "string",
        "required": True,
        "min_length": 2,
        "max_length": 100
    },
    "email": {
        "type": "email",
        "required": True
    },
    "amount": {
        "type": "numeric",
        "required": True,
        "min": 0
    }
})
def create_customer_secure(customer_name=None, email=None, amount=None):
    """Secure customer creation with input validation"""
    
    # Input is already validated and sanitized
    customer = frappe.get_doc({
        "doctype": "Customer",
        "customer_name": customer_name,
        "email_id": email
    })
    
    customer.insert()
    return customer.as_dict()

# Security Headers Middleware
def set_security_headers():
    """Set security headers for all responses"""
    
    security_headers = {
        "X-Content-Type-Options": "nosniff",
        "X-Frame-Options": "DENY",
        "X-XSS-Protection": "1; mode=block",
        "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
        "Referrer-Policy": "strict-origin-when-cross-origin",
        "Permissions-Policy": "geolocation=(), microphone=(), camera=()"
    }
    
    frappe.local.response.headers.update(security_headers)
    
    # Set CSP headers
    csp_manager = CSPManager()
    csp_manager.set_csp_headers()
```

This comprehensive security guide provides enterprise-grade security implementation patterns for protecting Frappe applications against common vulnerabilities while maintaining usability and performance.

**Key Features:**

1. **Authentication & Authorization** - Role-based access control, MFA, password security
2. **Input Validation** - Comprehensive validation, sanitization, XSS prevention
3. **SQL Injection Prevention** - Parameterized queries, input validation
4. **File Upload Security** - Content validation, type checking, malware detection
5. **Security Headers** - CSP, CSRF protection, security headers
6. **Audit Logging** - Security event logging, suspicious activity detection

**Usage Guidelines:**
- Always validate and sanitize user input at application boundaries
- Use parameterized queries to prevent SQL injection
- Implement proper authentication and authorization checks
- Set security headers for all responses
- Monitor and log security events for incident response
- Regularly update security configurations and dependencies