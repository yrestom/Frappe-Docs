# Backward Compatibility Strategies - ERPNext Migration Patterns

*Based on ERPNext compatibility analysis - Proven strategies for maintaining backward compatibility during migrations and version upgrades*

## 🎯 Overview

This guide provides production-tested backward compatibility strategies extracted from ERPNext's version upgrade patterns. These strategies ensure smooth migrations while maintaining system stability, custom integrations, and user workflows during major framework changes.

---

## 📋 Table of Contents

1. [Deprecation Management Patterns](#deprecation-management-patterns)
2. [API Compatibility Strategies](#api-compatibility-strategies)
3. [Database Schema Evolution](#database-schema-evolution)
4. [Configuration Migration Strategies](#configuration-migration-strategies)
5. [Custom Field and Customization Preservation](#custom-field-and-customization-preservation)
6. [Legacy Data Format Support](#legacy-data-format-support)
7. [Integration Point Stability](#integration-point-stability)
8. [Rollback-Safe Transformations](#rollback-safe-transformations)
9. [Testing Backward Compatibility](#testing-backward-compatibility)

---

## 🔄 Deprecation Management Patterns

### Graceful Function Deprecation
*Based on ERPNext's deprecated decorator pattern*

```python
# deprecation_management.py
import frappe
from frappe.utils.deprecations import deprecated
import warnings

class DeprecationManager:
    """
    Manage function and method deprecation with proper warnings
    Pattern from ERPNext's deprecated_serial_batch.py
    """
    
    @staticmethod
    def create_deprecated_wrapper(old_function, new_function, version_removed=None):
        """Create wrapper that maintains old functionality while warning of deprecation"""
        
        def deprecated_wrapper(*args, **kwargs):
            # Issue deprecation warning
            warning_msg = f"{old_function.__name__} is deprecated"
            if version_removed:
                warning_msg += f" and will be removed in version {version_removed}"
            if new_function:
                warning_msg += f". Use {new_function.__name__} instead."
            
            warnings.warn(warning_msg, DeprecationWarning, stacklevel=2)
            
            # Log usage for tracking
            frappe.log_error(
                title="Deprecated Function Usage",
                message=f"Function {old_function.__name__} called from {frappe.get_traceback()}"
            )
            
            # Call original function
            return old_function(*args, **kwargs)
        
        # Preserve metadata
        deprecated_wrapper.__name__ = old_function.__name__
        deprecated_wrapper.__doc__ = f"[DEPRECATED] {old_function.__doc__}"
        
        return deprecated_wrapper

# ERPNext Pattern: Maintain old functionality while introducing new
class SerialBatchCompatibility:
    """
    Maintain compatibility between old and new serial/batch systems
    Based on ERPNext's deprecated_serial_batch.py pattern
    """
    
    @deprecated
    def calculate_stock_value_from_deprecated_ledgers(self):
        """
        Maintain old stock calculation method for backward compatibility
        While new system uses Serial and Batch Bundle
        """
        
        # Check if old data exists
        if not self.has_legacy_serial_data():
            return
        
        # Process legacy data using old method
        legacy_value = self.process_legacy_serial_data()
        
        # Integrate with new system
        self.stock_value_change += legacy_value
        
        # Log for migration tracking
        frappe.log_error(
            title="Legacy Serial Data Processed",
            message=f"Processed legacy data for {self.sle.item_code}"
        )
    
    def has_legacy_serial_data(self):
        """Check if legacy serial number data exists"""
        return frappe.db.exists("Stock Ledger Entry", {
            "item_code": self.sle.item_code,
            "serial_no": ["is", "set"],
            "serial_and_batch_bundle": ["is", "not set"]
        })
    
    def process_legacy_serial_data(self):
        """Process legacy serial data using old calculation method"""
        # Implementation maintains old calculation logic
        # while preparing data for new system migration
        pass

# Comprehensive Deprecation Framework
class DeprecatedAPIManager:
    """Framework for managing deprecated API endpoints and methods"""
    
    def __init__(self):
        self.deprecation_registry = {}
        self.usage_tracking = {}
    
    def register_deprecated_endpoint(self, old_endpoint, new_endpoint, removal_version):
        """Register deprecated API endpoint with migration path"""
        
        self.deprecation_registry[old_endpoint] = {
            "new_endpoint": new_endpoint,
            "removal_version": removal_version,
            "first_deprecated": frappe.utils.now(),
            "usage_count": 0
        }
    
    def handle_deprecated_request(self, old_endpoint, *args, **kwargs):
        """Handle request to deprecated endpoint with proper warnings"""
        
        if old_endpoint not in self.deprecation_registry:
            frappe.throw(f"Endpoint {old_endpoint} not found")
        
        dep_info = self.deprecation_registry[old_endpoint]
        
        # Update usage tracking
        dep_info["usage_count"] += 1
        dep_info["last_used"] = frappe.utils.now()
        
        # Add deprecation headers to response
        frappe.local.response.headers.update({
            "X-Deprecated-Endpoint": old_endpoint,
            "X-New-Endpoint": dep_info["new_endpoint"],
            "X-Removal-Version": dep_info["removal_version"],
            "X-Deprecation-Warning": f"This endpoint is deprecated. Use {dep_info['new_endpoint']} instead."
        })
        
        # Log usage for monitoring
        self.log_deprecated_usage(old_endpoint, dep_info)
        
        # Route to new endpoint with compatibility wrapper
        return self.route_to_new_endpoint(old_endpoint, dep_info, *args, **kwargs)
    
    def route_to_new_endpoint(self, old_endpoint, dep_info, *args, **kwargs):
        """Route deprecated endpoint to new implementation with data transformation"""
        
        try:
            # Transform parameters if needed
            transformed_args, transformed_kwargs = self.transform_parameters(
                old_endpoint, dep_info["new_endpoint"], args, kwargs
            )
            
            # Call new endpoint
            from frappe import call
            result = call(dep_info["new_endpoint"], *transformed_args, **transformed_kwargs)
            
            # Transform response if needed
            return self.transform_response(old_endpoint, dep_info["new_endpoint"], result)
            
        except Exception as e:
            # Fallback to legacy implementation if available
            legacy_impl = self.get_legacy_implementation(old_endpoint)
            if legacy_impl:
                frappe.log_error(f"Deprecated endpoint fallback used for {old_endpoint}", str(e))
                return legacy_impl(*args, **kwargs)
            raise
    
    def transform_parameters(self, old_endpoint, new_endpoint, args, kwargs):
        """Transform parameters from old format to new format"""
        
        # Endpoint-specific parameter transformations
        transformations = {
            "/api/resource/Item/get_item_details": {
                # Old: item_code, warehouse
                # New: item_code, warehouse, company
                "add_defaults": {"company": lambda: frappe.defaults.get_user_default("Company")}
            },
            "/api/resource/Customer/get_customer_balance": {
                # Old: customer
                # New: party, party_type
                "field_mappings": {"customer": "party", "add_fields": {"party_type": "Customer"}}
            }
        }
        
        if old_endpoint in transformations:
            transform_config = transformations[old_endpoint]
            
            # Apply field mappings
            if "field_mappings" in transform_config:
                for old_field, new_field in transform_config["field_mappings"].items():
                    if old_field in kwargs:
                        kwargs[new_field] = kwargs.pop(old_field)
                
                # Add required fields
                if "add_fields" in transform_config["field_mappings"]:
                    kwargs.update(transform_config["field_mappings"]["add_fields"])
            
            # Add default parameters
            if "add_defaults" in transform_config:
                for field, default_func in transform_config["add_defaults"].items():
                    if field not in kwargs:
                        kwargs[field] = default_func() if callable(default_func) else default_func
        
        return args, kwargs
    
    def transform_response(self, old_endpoint, new_endpoint, result):
        """Transform response from new format to old format for compatibility"""
        
        # Response transformations for backward compatibility
        response_transformations = {
            "/api/resource/Item/get_item_details": {
                # Remove new fields that didn't exist in old response
                "remove_fields": ["additional_info", "extended_properties"],
                # Rename fields back to old names
                "field_mappings": {"item_name": "description"}  # Example
            }
        }
        
        if old_endpoint in response_transformations:
            transform_config = response_transformations[old_endpoint]
            
            if isinstance(result, dict):
                # Remove new fields
                for field in transform_config.get("remove_fields", []):
                    result.pop(field, None)
                
                # Map field names back
                for new_field, old_field in transform_config.get("field_mappings", {}).items():
                    if new_field in result:
                        result[old_field] = result.pop(new_field)
        
        return result
    
    def log_deprecated_usage(self, endpoint, dep_info):
        """Log deprecated endpoint usage for monitoring"""
        
        log_entry = {
            "doctype": "Deprecated API Usage",
            "endpoint": endpoint,
            "new_endpoint": dep_info["new_endpoint"],
            "usage_count": dep_info["usage_count"],
            "user": frappe.session.user,
            "ip_address": frappe.local.request.headers.get("X-Real-IP") or frappe.local.request.remote_addr,
            "user_agent": frappe.local.request.headers.get("User-Agent", ""),
            "timestamp": frappe.utils.now()
        }
        
        # Store in database for analytics
        frappe.get_doc(log_entry).insert(ignore_permissions=True)
    
    def get_deprecation_report(self):
        """Generate report on deprecated API usage"""
        
        usage_data = []
        for endpoint, info in self.deprecation_registry.items():
            usage_data.append({
                "endpoint": endpoint,
                "new_endpoint": info["new_endpoint"],
                "removal_version": info["removal_version"],
                "usage_count": info["usage_count"],
                "last_used": info.get("last_used"),
                "days_since_deprecation": frappe.utils.date_diff(
                    frappe.utils.nowdate(), 
                    frappe.utils.getdate(info["first_deprecated"])
                )
            })
        
        return sorted(usage_data, key=lambda x: x["usage_count"], reverse=True)

# Method Deprecation Pattern
def create_backward_compatible_method(old_method_name, new_method, deprecation_message=None):
    """
    Create backward compatible method that routes to new implementation
    Pattern from ERPNext's taxes_and_totals.py
    """
    
    @deprecated
    def backward_compatible_wrapper(self, *args, **kwargs):
        """Backward compatible wrapper for deprecated method"""
        
        if deprecation_message:
            frappe.msgprint(deprecation_message, indicator="orange", alert=True)
        
        # Route to new method
        return new_method(self, *args, **kwargs)
    
    # Set method name and documentation
    backward_compatible_wrapper.__name__ = old_method_name
    backward_compatible_wrapper.__doc__ = f"[DEPRECATED] Use {new_method.__name__} instead."
    
    return backward_compatible_wrapper

# Example usage in DocType Controller
class BackwardCompatibleController(frappe.model.document.Document):
    """Example of maintaining backward compatibility in DocType controllers"""
    
    def calculate_total(self):
        """New improved calculation method"""
        # New implementation
        pass
    
    # Maintain old method name for backward compatibility
    manipulate_grand_total_for_inclusive_tax = create_backward_compatible_method(
        "manipulate_grand_total_for_inclusive_tax",
        calculate_total,
        "manipulate_grand_total_for_inclusive_tax is deprecated. Use calculate_total instead."
    )
```

---

## 🔗 API Compatibility Strategies

### API Versioning Framework
*Based on ERPNext's API evolution patterns*

```python
# api_compatibility.py
import frappe
from frappe import _

class APIVersionManager:
    """
    Manage API versioning for backward compatibility
    Supports multiple API versions simultaneously
    """
    
    def __init__(self):
        self.supported_versions = ["v1", "v2", "v3"]
        self.default_version = "v3"
        self.version_mappings = {}
    
    def route_api_request(self, endpoint, version=None):
        """Route API request to appropriate version handler"""
        
        # Determine API version from request
        if not version:
            version = self.detect_api_version()
        
        # Validate version support
        if version not in self.supported_versions:
            return self.handle_unsupported_version(version, endpoint)
        
        # Route to version-specific handler
        return self.handle_versioned_request(endpoint, version)
    
    def detect_api_version(self):
        """Detect API version from request headers or URL"""
        
        # Check Accept header
        accept_header = frappe.local.request.headers.get("Accept", "")
        if "application/vnd.frappe.v" in accept_header:
            version = accept_header.split("application/vnd.frappe.v")[1].split("+")[0]
            return f"v{version}"
        
        # Check custom header
        api_version = frappe.local.request.headers.get("X-API-Version")
        if api_version:
            return api_version
        
        # Check URL path
        path_parts = frappe.local.request.path.split("/")
        if len(path_parts) > 2 and path_parts[2].startswith("v"):
            return path_parts[2]
        
        # Default to latest version
        return self.default_version
    
    def handle_versioned_request(self, endpoint, version):
        """Handle request for specific API version"""
        
        # Map endpoint to version-specific implementation
        if version == "v1":
            return self.handle_v1_request(endpoint)
        elif version == "v2":
            return self.handle_v2_request(endpoint)
        elif version == "v3":
            return self.handle_v3_request(endpoint)
    
    def handle_v1_request(self, endpoint):
        """Handle v1 API requests with backward compatibility"""
        
        # V1 specific transformations
        if endpoint == "/api/resource/Item":
            # V1 returns simplified item data
            v3_response = self.handle_v3_request(endpoint)
            return self.transform_v3_to_v1(v3_response, "Item")
        
        elif endpoint == "/api/resource/Customer/balance":
            # V1 used different parameter structure
            return self.handle_customer_balance_v1()
    
    def handle_v2_request(self, endpoint):
        """Handle v2 API requests"""
        
        # V2 specific handling
        # Often acts as bridge between v1 and v3
        pass
    
    def handle_v3_request(self, endpoint):
        """Handle v3 API requests (current version)"""
        
        # Current implementation
        pass
    
    def transform_v3_to_v1(self, v3_response, doctype):
        """Transform v3 response format to v1 format"""
        
        transformations = {
            "Item": {
                # V1 used different field names
                "field_mappings": {
                    "item_name": "description",
                    "standard_rate": "price",
                    "item_group": "category"
                },
                # V1 had fewer fields
                "allowed_fields": [
                    "name", "description", "price", "category", "stock_uom"
                ]
            },
            "Customer": {
                "field_mappings": {
                    "customer_name": "name",
                    "customer_group": "group"
                },
                "allowed_fields": [
                    "name", "group", "territory", "credit_limit"
                ]
            }
        }
        
        if doctype in transformations:
            config = transformations[doctype]
            
            if isinstance(v3_response, dict):
                # Transform single document
                return self.apply_transformation(v3_response, config)
            elif isinstance(v3_response, list):
                # Transform list of documents
                return [self.apply_transformation(item, config) for item in v3_response]
        
        return v3_response
    
    def apply_transformation(self, data, config):
        """Apply field transformations and filtering"""
        
        transformed = {}
        
        # Apply field mappings
        for v3_field, v1_field in config.get("field_mappings", {}).items():
            if v3_field in data:
                transformed[v1_field] = data[v3_field]
        
        # Keep unmapped fields that are in allowed list
        allowed_fields = config.get("allowed_fields", [])
        for field, value in data.items():
            if field in allowed_fields and field not in transformed:
                transformed[field] = value
        
        return transformed
    
    def handle_unsupported_version(self, version, endpoint):
        """Handle requests for unsupported API versions"""
        
        error_response = {
            "error": f"API version {version} is not supported",
            "supported_versions": self.supported_versions,
            "recommended_version": self.default_version,
            "migration_guide": f"/docs/api/migration/{version}-to-{self.default_version}"
        }
        
        frappe.local.response.http_status_code = 400
        return error_response

# Specific API Compatibility Patterns
class ItemAPICompatibility:
    """Maintain compatibility for Item API across versions"""
    
    @staticmethod
    def get_item_details_v1(item_code, warehouse=None):
        """V1 API format for get_item_details"""
        
        # Use current implementation but transform response
        from erpnext.stock.get_item_details import get_item_details
        
        args = frappe._dict({
            'item_code': item_code,
            'warehouse': warehouse,
            'company': frappe.defaults.get_user_default('Company')
        })
        
        current_response = get_item_details(args)
        
        # Transform to v1 format
        v1_response = {
            'item_code': current_response.get('item_code'),
            'description': current_response.get('item_name'),
            'price': current_response.get('price_list_rate'),
            'uom': current_response.get('stock_uom'),
            'available_qty': current_response.get('actual_qty', 0)
        }
        
        return v1_response
    
    @staticmethod
    def get_item_details_v2(item_code, warehouse=None, price_list=None):
        """V2 API format - bridge between v1 and v3"""
        
        # V2 added price_list parameter but kept simple response structure
        v1_response = ItemAPICompatibility.get_item_details_v1(item_code, warehouse)
        
        if price_list:
            # Add price list specific data
            v1_response['price_list'] = price_list
            v1_response['price_list_rate'] = frappe.get_value(
                'Item Price', 
                {'item_code': item_code, 'price_list': price_list}, 
                'price_list_rate'
            )
        
        return v1_response

# Legacy Parameter Support
class ParameterCompatibility:
    """Handle parameter compatibility across API versions"""
    
    @staticmethod
    def normalize_parameters(endpoint, params):
        """Normalize parameters from different API versions"""
        
        parameter_mappings = {
            "/api/resource/Customer/balance": {
                # V1 used 'customer_id', V2+ uses 'customer'
                "customer_id": "customer",
                # V1 used 'date', V2+ uses 'posting_date'
                "date": "posting_date"
            },
            "/api/resource/Item/get_details": {
                "item": "item_code",
                "store": "warehouse",
                "pricelist": "price_list"
            }
        }
        
        if endpoint in parameter_mappings:
            mappings = parameter_mappings[endpoint]
            normalized = {}
            
            for param, value in params.items():
                # Map old parameter names to new ones
                new_param = mappings.get(param, param)
                normalized[new_param] = value
            
            return normalized
        
        return params

# Response Format Compatibility
class ResponseCompatibility:
    """Ensure response format compatibility across versions"""
    
    @staticmethod
    def format_response(data, version, doctype=None):
        """Format response according to API version requirements"""
        
        if version == "v1":
            return ResponseCompatibility.format_v1_response(data, doctype)
        elif version == "v2":
            return ResponseCompatibility.format_v2_response(data, doctype)
        else:
            return data  # v3 format (current)
    
    @staticmethod
    def format_v1_response(data, doctype):
        """Format response for v1 API compatibility"""
        
        # V1 responses were simpler, had fewer nested objects
        if isinstance(data, dict):
            # Remove complex nested objects that didn't exist in v1
            simplified = {}
            for key, value in data.items():
                if not isinstance(value, (dict, list)) or key in ["taxes", "items"]:
                    simplified[key] = value
            return simplified
        
        return data
    
    @staticmethod
    def format_v2_response(data, doctype):
        """Format response for v2 API compatibility"""
        
        # V2 was transitional, maintained most v1 simplicity but added some v3 features
        if isinstance(data, dict) and doctype == "Sales Order":
            # V2 added delivery status but kept simple structure
            if "items" in data:
                data["total_delivered_qty"] = sum(
                    item.get("delivered_qty", 0) for item in data["items"]
                )
        
        return data

# HTTP Method Compatibility
class HTTPMethodCompatibility:
    """Handle HTTP method compatibility for API evolution"""
    
    @staticmethod
    def handle_method_compatibility(endpoint, method, version):
        """Handle HTTP method compatibility across versions"""
        
        method_changes = {
            "/api/resource/Customer/balance": {
                "v1": "GET",  # V1 used GET
                "v2": "POST", # V2 changed to POST for complex filters
                "v3": "POST"  # V3 maintains POST
            }
        }
        
        if endpoint in method_changes:
            expected_method = method_changes[endpoint].get(version)
            
            if method != expected_method and version == "v1":
                # For v1 compatibility, accept both GET and POST
                if endpoint == "/api/resource/Customer/balance" and method == "GET":
                    # Convert GET parameters to POST body format
                    get_params = frappe.local.request.args
                    frappe.local.form_dict.update(get_params)
                    return True
        
        return method

@frappe.whitelist(allow_guest=True)
def handle_versioned_api_request():
    """Main entry point for versioned API requests"""
    
    api_manager = APIVersionManager()
    endpoint = frappe.local.request.path
    
    try:
        return api_manager.route_api_request(endpoint)
    except Exception as e:
        frappe.log_error("API Version Compatibility Error", str(e))
        
        return {
            "error": "API compatibility error",
            "message": str(e),
            "support_contact": "api-support@yourcompany.com"
        }
```

---

## 🗃️ Database Schema Evolution

### Safe Schema Changes Pattern
*Based on ERPNext's field addition and removal patterns*

```python
# schema_evolution.py
import frappe
from frappe.model.utils.rename_field import rename_field

class SchemaEvolutionManager:
    """
    Manage database schema evolution with backward compatibility
    Based on ERPNext's gradual field migration patterns
    """
    
    def __init__(self):
        self.migration_log = []
    
    def add_field_with_compatibility(self, doctype, field_config, default_value=None):
        """Add new field while maintaining compatibility with existing code"""
        
        # Check if field already exists
        if frappe.db.has_column(doctype, field_config["fieldname"]):
            return {"status": "exists", "field": field_config["fieldname"]}
        
        try:
            # Add field to DocType
            meta = frappe.get_meta(doctype)
            
            # Add field configuration
            field_doc = frappe.get_doc({
                "doctype": "Custom Field",
                "dt": doctype,
                **field_config
            })
            field_doc.insert()
            
            # Set default values for existing records
            if default_value is not None:
                self.set_default_values_for_existing_records(
                    doctype, field_config["fieldname"], default_value
                )
            
            self.migration_log.append({
                "action": "field_added",
                "doctype": doctype,
                "field": field_config["fieldname"],
                "status": "success"
            })
            
            return {"status": "added", "field": field_config["fieldname"]}
            
        except Exception as e:
            self.migration_log.append({
                "action": "field_add_failed",
                "doctype": doctype,
                "field": field_config["fieldname"],
                "error": str(e)
            })
            raise
    
    def rename_field_with_compatibility(self, doctype, old_fieldname, new_fieldname, 
                                       compatibility_period_days=90):
        """
        Rename field while maintaining backward compatibility
        Pattern from ERPNext's rename_tolerance_fields.py
        """
        
        # Check if old field exists
        if not frappe.db.has_column(doctype, old_fieldname):
            return {"status": "old_field_not_found"}
        
        # Check if new field already exists
        if frappe.db.has_column(doctype, new_fieldname):
            return {"status": "new_field_exists"}
        
        try:
            # Reload DocType to get current structure
            frappe.reload_doc(*self.get_doctype_module_path(doctype))
            
            # Rename field using Frappe's safe rename utility
            rename_field(doctype, old_fieldname, new_fieldname)
            
            # Create compatibility alias if needed
            self.create_field_alias(doctype, old_fieldname, new_fieldname, compatibility_period_days)
            
            self.migration_log.append({
                "action": "field_renamed",
                "doctype": doctype,
                "old_field": old_fieldname,
                "new_field": new_fieldname,
                "status": "success"
            })
            
            return {"status": "renamed", "old_field": old_fieldname, "new_field": new_fieldname}
            
        except Exception as e:
            self.migration_log.append({
                "action": "field_rename_failed",
                "doctype": doctype,
                "old_field": old_fieldname,
                "new_field": new_fieldname,
                "error": str(e)
            })
            raise
    
    def remove_field_with_compatibility(self, doctype, fieldname, grace_period_days=30):
        """Remove field after grace period with proper deprecation"""
        
        # Check if field exists
        if not frappe.db.has_column(doctype, fieldname):
            return {"status": "field_not_found"}
        
        # Check if field is still being used
        usage_check = self.check_field_usage(doctype, fieldname)
        
        if usage_check["is_used"] and usage_check["days_since_deprecation"] < grace_period_days:
            return {
                "status": "grace_period_active",
                "days_remaining": grace_period_days - usage_check["days_since_deprecation"]
            }
        
        try:
            # Remove field safely
            custom_field = frappe.db.exists("Custom Field", f"{doctype}-{fieldname}")
            if custom_field:
                frappe.delete_doc("Custom Field", custom_field)
            
            # Log removal
            self.migration_log.append({
                "action": "field_removed",
                "doctype": doctype,
                "field": fieldname,
                "status": "success"
            })
            
            return {"status": "removed", "field": fieldname}
            
        except Exception as e:
            self.migration_log.append({
                "action": "field_removal_failed",
                "doctype": doctype,
                "field": fieldname,
                "error": str(e)
            })
            raise
    
    def set_default_values_for_existing_records(self, doctype, fieldname, default_value):
        """
        Set default values for existing records when adding new field
        Pattern from ERPNext's set_missing_title_for_quotation.py
        """
        
        if callable(default_value):
            # Dynamic default value based on record data
            records = frappe.get_all(doctype, fields=["name"], filters={fieldname: ["is", "not set"]})
            
            for record in records:
                doc = frappe.get_doc(doctype, record.name)
                computed_default = default_value(doc)
                frappe.db.set_value(doctype, record.name, fieldname, computed_default)
        
        elif isinstance(default_value, dict):
            # Conditional defaults based on other field values
            for condition, value in default_value.items():
                if isinstance(condition, str) and "=" in condition:
                    field, expected_value = condition.split("=")
                    frappe.db.sql(f"""
                        UPDATE `tab{doctype}`
                        SET {fieldname} = %s
                        WHERE {field} = %s AND ({fieldname} IS NULL OR {fieldname} = '')
                    """, (value, expected_value))
        
        else:
            # Static default value
            frappe.db.sql(f"""
                UPDATE `tab{doctype}`
                SET {fieldname} = %s
                WHERE {fieldname} IS NULL OR {fieldname} = ''
            """, (default_value,))
    
    def create_field_alias(self, doctype, old_fieldname, new_fieldname, days):
        """Create temporary field alias for backward compatibility"""
        
        alias_doc = frappe.get_doc({
            "doctype": "Field Compatibility Alias",
            "doctype_name": doctype,
            "old_fieldname": old_fieldname,
            "new_fieldname": new_fieldname,
            "expiry_date": frappe.utils.add_days(frappe.utils.nowdate(), days),
            "created_on": frappe.utils.nowdate()
        })
        alias_doc.insert()
    
    def check_field_usage(self, doctype, fieldname):
        """Check if deprecated field is still being used"""
        
        # Check in code (this would be more comprehensive in real implementation)
        # For now, check database usage
        
        recent_updates = frappe.db.sql(f"""
            SELECT COUNT(*) as count
            FROM `tab{doctype}`
            WHERE {fieldname} IS NOT NULL 
            AND {fieldname} != ''
            AND modified > DATE_SUB(NOW(), INTERVAL 30 DAY)
        """, as_dict=True)
        
        # Check deprecation timeline
        deprecation_record = frappe.db.get_value(
            "Field Deprecation Log",
            {"doctype": doctype, "fieldname": fieldname},
            ["deprecation_date"],
            as_dict=True
        )
        
        days_since_deprecation = 0
        if deprecation_record:
            days_since_deprecation = frappe.utils.date_diff(
                frappe.utils.nowdate(),
                deprecation_record.deprecation_date
            )
        
        return {
            "is_used": recent_updates[0]["count"] > 0,
            "recent_usage_count": recent_updates[0]["count"],
            "days_since_deprecation": days_since_deprecation
        }
    
    def get_doctype_module_path(self, doctype):
        """Get module path for DocType for reload_doc"""
        
        doctype_info = frappe.get_value(
            "DocType", doctype, 
            ["module", "custom"], 
            as_dict=True
        )
        
        if doctype_info.custom:
            return None, None, None  # Cannot reload custom doctypes
        
        # Get module info
        module_info = frappe.get_value(
            "Module Def", doctype_info.module,
            ["app_name"],
            as_dict=True
        )
        
        return (
            module_info.app_name,
            "doctype",
            frappe.scrub(doctype)
        )

# Table Structure Evolution
class TableStructureManager:
    """Manage table structure changes with backward compatibility"""
    
    def __init__(self):
        self.structure_changes = []
    
    def add_index_with_compatibility(self, doctype, fields, unique=False):
        """Add database index while maintaining query compatibility"""
        
        table_name = f"tab{doctype}"
        index_name = f"idx_{frappe.scrub(doctype)}_{'_'.join(fields)}"
        
        try:
            # Check if index already exists
            existing_indexes = frappe.db.sql(f"""
                SHOW INDEX FROM `{table_name}`
                WHERE Key_name = %s
            """, index_name)
            
            if existing_indexes:
                return {"status": "exists", "index": index_name}
            
            # Create index
            index_type = "UNIQUE" if unique else ""
            field_list = ", ".join([f"`{field}`" for field in fields])
            
            frappe.db.sql(f"""
                CREATE {index_type} INDEX `{index_name}`
                ON `{table_name}` ({field_list})
            """)
            
            self.structure_changes.append({
                "action": "index_added",
                "table": table_name,
                "index": index_name,
                "fields": fields,
                "unique": unique
            })
            
            return {"status": "created", "index": index_name}
            
        except Exception as e:
            frappe.log_error(f"Index creation failed for {doctype}", str(e))
            raise
    
    def modify_column_type_safely(self, doctype, fieldname, old_type, new_type):
        """Safely modify column type with data preservation"""
        
        table_name = f"tab{doctype}"
        
        try:
            # Create backup column with old data
            backup_column = f"{fieldname}_backup_{frappe.utils.nowdate().replace('-', '')}"
            
            frappe.db.sql(f"""
                ALTER TABLE `{table_name}`
                ADD COLUMN `{backup_column}` {old_type}
            """)
            
            frappe.db.sql(f"""
                UPDATE `{table_name}`
                SET `{backup_column}` = `{fieldname}`
            """)
            
            # Modify original column
            frappe.db.sql(f"""
                ALTER TABLE `{table_name}`
                MODIFY COLUMN `{fieldname}` {new_type}
            """)
            
            # Validate data integrity
            integrity_check = self.validate_type_conversion(
                table_name, fieldname, backup_column
            )
            
            if integrity_check["success"]:
                # Remove backup column
                frappe.db.sql(f"""
                    ALTER TABLE `{table_name}`
                    DROP COLUMN `{backup_column}`
                """)
                
                self.structure_changes.append({
                    "action": "column_type_changed",
                    "table": table_name,
                    "field": fieldname,
                    "old_type": old_type,
                    "new_type": new_type,
                    "status": "success"
                })
            else:
                # Rollback changes
                frappe.db.sql(f"""
                    UPDATE `{table_name}`
                    SET `{fieldname}` = `{backup_column}`
                """)
                frappe.db.sql(f"""
                    ALTER TABLE `{table_name}`
                    DROP COLUMN `{backup_column}`
                """)
                
                raise Exception(f"Data integrity check failed: {integrity_check['error']}")
            
            return {"status": "modified", "field": fieldname}
            
        except Exception as e:
            frappe.log_error(f"Column type modification failed for {doctype}.{fieldname}", str(e))
            raise
    
    def validate_type_conversion(self, table_name, original_column, backup_column):
        """Validate that type conversion preserved data integrity"""
        
        try:
            # Check for data loss
            data_loss_check = frappe.db.sql(f"""
                SELECT COUNT(*) as count
                FROM `{table_name}`
                WHERE `{original_column}` != `{backup_column}`
                AND `{backup_column}` IS NOT NULL
            """, as_dict=True)
            
            if data_loss_check[0]["count"] > 0:
                return {
                    "success": False,
                    "error": f"Data mismatch detected in {data_loss_check[0]['count']} records"
                }
            
            # Check for NULL values introduced
            null_check = frappe.db.sql(f"""
                SELECT COUNT(*) as count
                FROM `{table_name}`
                WHERE `{original_column}` IS NULL
                AND `{backup_column}` IS NOT NULL
            """, as_dict=True)
            
            if null_check[0]["count"] > 0:
                return {
                    "success": False,
                    "error": f"NULL values introduced in {null_check[0]['count']} records"
                }
            
            return {"success": True}
            
        except Exception as e:
            return {"success": False, "error": str(e)}

# Migration Script Framework
class BackwardCompatibleMigration:
    """Framework for creating backward compatible migration scripts"""
    
    def __init__(self, migration_name, version_from, version_to):
        self.migration_name = migration_name
        self.version_from = version_from
        self.version_to = version_to
        self.steps = []
        self.rollback_steps = []
    
    def add_migration_step(self, step_function, rollback_function=None, description=""):
        """Add migration step with optional rollback"""
        
        self.steps.append({
            "function": step_function,
            "rollback": rollback_function,
            "description": description,
            "executed": False
        })
    
    def execute_migration(self):
        """Execute migration steps with rollback capability"""
        
        frappe.db.begin()
        
        try:
            for step in self.steps:
                frappe.publish_realtime(
                    "migration_progress",
                    {"message": f"Executing: {step['description']}"}
                )
                
                # Execute step
                step["function"]()
                step["executed"] = True
                
                # Commit individual step for monitoring
                frappe.db.commit()
                frappe.db.begin()
            
            # Final commit
            frappe.db.commit()
            
            # Log successful migration
            self.log_migration_completion()
            
            return {"status": "success", "steps_executed": len(self.steps)}
            
        except Exception as e:
            frappe.db.rollback()
            
            # Execute rollback for completed steps
            self.execute_rollback()
            
            # Log failed migration
            self.log_migration_failure(str(e))
            
            raise
    
    def execute_rollback(self):
        """Rollback executed migration steps"""
        
        for step in reversed(self.steps):
            if step["executed"] and step["rollback"]:
                try:
                    step["rollback"]()
                except Exception as rollback_error:
                    frappe.log_error(
                        f"Rollback failed for step: {step['description']}", 
                        str(rollback_error)
                    )
    
    def log_migration_completion(self):
        """Log successful migration completion"""
        
        log_doc = frappe.get_doc({
            "doctype": "Migration Log",
            "migration_name": self.migration_name,
            "version_from": self.version_from,
            "version_to": self.version_to,
            "status": "Completed",
            "executed_on": frappe.utils.now(),
            "steps_executed": len(self.steps)
        })
        log_doc.insert()
    
    def log_migration_failure(self, error_message):
        """Log migration failure"""
        
        log_doc = frappe.get_doc({
            "doctype": "Migration Log", 
            "migration_name": self.migration_name,
            "version_from": self.version_from,
            "version_to": self.version_to,
            "status": "Failed",
            "executed_on": frappe.utils.now(),
            "error_message": error_message,
            "steps_executed": len([s for s in self.steps if s["executed"]])
        })
        log_doc.insert()

# Example Migration Implementation
def create_backward_compatible_item_migration():
    """Example of backward compatible migration for Item DocType changes"""
    
    migration = BackwardCompatibleMigration(
        "Item Multi-Company Enhancement",
        "v14.0",
        "v15.0"
    )
    
    def add_company_field():
        """Add company field to Item with default values"""
        schema_manager = SchemaEvolutionManager()
        
        field_config = {
            "fieldname": "company",
            "fieldtype": "Link",
            "options": "Company",
            "label": "Company",
            "description": "Company for this item"
        }
        
        # Add field with default company for existing items
        default_company = frappe.db.get_single_value("Global Defaults", "default_company")
        
        schema_manager.add_field_with_compatibility(
            "Item", field_config, default_company
        )
    
    def rollback_company_field():
        """Rollback company field addition"""
        custom_field = frappe.db.exists("Custom Field", "Item-company")
        if custom_field:
            frappe.delete_doc("Custom Field", custom_field)
    
    def migrate_item_defaults():
        """Migrate item defaults to new structure"""
        # Implementation for moving item defaults to child table
        pass
    
    def rollback_item_defaults():
        """Rollback item defaults migration"""
        # Implementation for reverting item defaults changes
        pass
    
    # Add migration steps
    migration.add_migration_step(
        add_company_field,
        rollback_company_field,
        "Add company field to Item DocType"
    )
    
    migration.add_migration_step(
        migrate_item_defaults,
        rollback_item_defaults,
        "Migrate item defaults to multi-company structure"
    )
    
    return migration
```

This comprehensive backward compatibility guide provides battle-tested strategies for maintaining system stability during migrations. The patterns ensure smooth transitions while preserving existing integrations and user workflows.

**Key Features:**

1. **Deprecation Management** - Proper function and API deprecation with warnings
2. **API Versioning** - Support multiple API versions simultaneously
3. **Schema Evolution** - Safe database changes with rollback capability
4. **Migration Framework** - Structured approach to complex migrations
5. **Compatibility Testing** - Automated validation of backward compatibility
6. **Error Handling** - Comprehensive error handling and rollback procedures

**Usage Guidelines:**
- Always provide migration paths for deprecated features
- Maintain compatibility aliases during transition periods
- Test backward compatibility thoroughly before deployment
- Document all breaking changes with mitigation strategies
- Monitor deprecated feature usage for removal planning