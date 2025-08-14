# Client-Side Development - Complete Guide

> **Comprehensive reference for JavaScript frontend development in Frappe Framework**

## Table of Contents

- [JavaScript Framework Architecture](#javascript-framework-architecture)
- [Form Scripts & Events](#form-scripts--events)
- [Field Controls & Customization](#field-controls--customization)
- [List View Customization](#list-view-customization)
- [Report Development](#report-development)
- [Dashboard & Workspace Creation](#dashboard--workspace-creation)
- [Dialog & Modal Management](#dialog--modal-management)
- [Client-Server Communication](#client-server-communication)
- [UI Components & Widgets](#ui-components--widgets)
- [Advanced JavaScript Patterns](#advanced-javascript-patterns)

## JavaScript Framework Architecture

### Client Framework Overview

Based on analysis of `frappe/public/js/frappe/form/form.js`, the Frappe client framework provides:

**Core Components:**
- **Form Framework**: Dynamic form generation and management
- **List Views**: Configurable data grids with filtering and sorting
- **Report Builder**: Interactive report generation
- **Dashboard System**: Widget-based dashboards
- **Real-time Updates**: WebSocket-based live updates
- **Router System**: Single-page application routing

### Form Controller Structure

```javascript
// From frappe/public/js/frappe/form/form.js:18-22
frappe.ui.form.Controller = class FormController {
    constructor(opts) {
        $.extend(this, opts);
    }
};

// FrappeForm class initialization
frappe.ui.form.Form = class FrappeForm {
    constructor(doctype, parent, in_form, doctype_layout_name) {
        this.docname = "";
        this.doctype = doctype;
        this.custom_buttons = {};
        this.sections = [];
        this.grids = [];
        this.events = {};
        // ... additional initialization
    }
};
```

## Form Scripts & Events

### Basic Form Event Pattern

**Document-Level Events:**
```javascript
// apps/library_management/library_management/doctype/library_member/library_member.js
frappe.ui.form.on('Library Member', {
    // Form Setup
    setup: function(frm) {
        // Called once when form is first loaded
        // Set up queries, filters, and form configuration
        frm.set_query("branch", function() {
            return {
                "filters": {
                    "is_active": 1,
                    "branch_type": "Library"
                }
            };
        });
        
        // Set up custom formatters
        frm.set_currency_labels(["membership_fee"], "USD");
    },
    
    onload: function(frm) {
        // Called every time form is loaded
        frm.set_value('registration_date', frappe.datetime.get_today());
        
        // Load related data
        if (frm.doc.member_id) {
            frm.trigger('load_member_statistics');
        }
    },
    
    refresh: function(frm) {
        // Called after form is loaded and after every save
        frm.toggle_display('membership_expiry', frm.doc.membership_type !== 'Lifetime');
        
        // Custom buttons based on document state
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__('Generate ID Card'), function() {
                frm.trigger('generate_id_card');
            }, __('Actions'));
            
            frm.add_custom_button(__('Suspend Membership'), function() {
                frm.trigger('suspend_membership');
            }, __('Actions'));
        }
        
        // Dynamic field properties
        frm.set_df_property('email', 'read_only', frm.doc.is_verified);
        
        // Show/hide sections based on data
        frm.toggle_display('emergency_contact_section', frm.doc.membership_type === 'Premium');
    },
    
    // Field Events
    membership_type: function(frm) {
        // Calculate membership fee based on type
        if (frm.doc.membership_type) {
            frappe.call({
                method: 'library_management.api.get_membership_fee',
                args: {
                    'membership_type': frm.doc.membership_type,
                    'member_age': frm.doc.age
                },
                callback: function(r) {
                    if (r.message) {
                        frm.set_value('membership_fee', r.message);
                    }
                }
            });
        }
    },
    
    email: function(frm) {
        // Email validation and duplicate check
        if (frm.doc.email) {
            if (!frappe.utils.validate_type(frm.doc.email, 'email')) {
                frappe.msgprint(__('Please enter a valid email address'));
                frm.set_value('email', '');
                return;
            }
            
            // Check for duplicate email
            frappe.call({
                method: 'frappe.client.get_list',
                args: {
                    doctype: 'Library Member',
                    filters: {
                        email: frm.doc.email,
                        name: ['!=', frm.doc.name || 'new-library-member-1']
                    },
                    limit_page_length: 1
                },
                callback: function(r) {
                    if (r.message && r.message.length > 0) {
                        frappe.msgprint(__('A member with this email already exists'));
                        frm.set_value('email', '');
                    }
                }
            });
        }
    },
    
    // Custom Methods
    generate_id_card: function(frm) {
        frappe.call({
            method: 'library_management.api.generate_member_id_card',
            args: {
                'member': frm.doc.name
            },
            callback: function(r) {
                if (r.message && r.message.file_url) {
                    window.open(r.message.file_url);
                    frappe.msgprint(__('ID Card generated successfully'));
                }
            }
        });
    },
    
    load_member_statistics: function(frm) {
        frappe.call({
            method: 'library_management.api.get_member_dashboard',
            args: {
                'member': frm.doc.name
            },
            callback: function(r) {
                if (r.message) {
                    // Update dashboard section
                    frm.dashboard.add_indicator(__('Books Issued: {0}', [r.message.current_issues_count]), 
                        r.message.current_issues_count > 0 ? 'blue' : 'green');
                    frm.dashboard.add_indicator(__('Overdue Books: {0}', [r.message.overdue_count]), 
                        r.message.overdue_count > 0 ? 'red' : 'green');
                }
            }
        });
    },
    
    // Validation
    validate: function(frm) {
        // Client-side validation
        if (frm.doc.membership_type === 'Student' && !frm.doc.student_id) {
            frappe.msgprint(__('Student ID is required for student membership'));
            frappe.validated = false;
        }
        
        if (frm.doc.birth_date && frappe.datetime.get_diff(frappe.datetime.get_today(), frm.doc.birth_date) < 0) {
            frappe.msgprint(__('Birth date cannot be in the future'));
            frappe.validated = false;
        }
    },
    
    // Before Save
    before_save: function(frm) {
        // Calculate age from birth date
        if (frm.doc.birth_date) {
            const age = moment().diff(moment(frm.doc.birth_date), 'years');
            frm.set_value('age', age);
        }
    },
    
    // After Save
    after_save: function(frm) {
        // Actions after successful save
        if (frm.doc.__islocal !== 1) {  // Not a new document
            frappe.show_alert({
                message: __('Member information updated successfully'),
                indicator: 'green'
            });
        }
    }
});
```

### Child Table Events

**Child Table Pattern:**
```javascript
// Child table: Contact Details
frappe.ui.form.on('Contact Detail', {
    // When a row is added
    contact_details_add: function(frm, cdt, cdn) {
        var row = locals[cdt][cdn];
        
        // Set default values
        row.contact_type = 'Phone';
        row.is_primary = 0;
        
        // Refresh the field
        frm.refresh_field('contact_details');
    },
    
    // When contact type changes
    contact_type: function(frm, cdt, cdn) {
        var row = locals[cdt][cdn];
        
        // Set field properties based on contact type
        if (row.contact_type === 'Email') {
            // Validate email format
            frappe.meta.get_docfield('Contact Detail', 'contact_value', row.name).options = 'Email';
        } else if (row.contact_type === 'Phone') {
            frappe.meta.get_docfield('Contact Detail', 'contact_value', row.name).options = 'Phone';
        }
        
        frm.refresh_field('contact_details');
    },
    
    // When row is removed
    contact_details_remove: function(frm, cdt, cdn) {
        // Recalculate something or update parent
        frm.trigger('calculate_contact_summary');
    },
    
    // Field validation in child table
    contact_value: function(frm, cdt, cdn) {
        var row = locals[cdt][cdn];
        
        if (row.contact_type === 'Email' && row.contact_value) {
            if (!frappe.utils.validate_type(row.contact_value, 'email')) {
                frappe.msgprint(__('Row {0}: Please enter a valid email address', [row.idx]));
                frappe.model.set_value(cdt, cdn, 'contact_value', '');
            }
        }
    }
});
```

## Field Controls & Customization

### Dynamic Field Control

**Field Manipulation Methods:**
```javascript
frappe.ui.form.on('Library Member', {
    refresh: function(frm) {
        // Show/Hide Fields
        frm.toggle_display('student_id', frm.doc.membership_type === 'Student');
        frm.toggle_display(['emergency_contact', 'emergency_phone'], frm.doc.age < 18);
        
        // Enable/Disable Fields
        frm.toggle_enable('email', !frm.doc.is_verified);
        frm.set_df_property('membership_fee', 'read_only', frm.doc.docstatus === 1);
        
        // Required Fields
        frm.toggle_reqd('student_id', frm.doc.membership_type === 'Student');
        frm.toggle_reqd(['emergency_contact', 'emergency_phone'], frm.doc.age < 18);
        
        // Field Labels
        frm.set_df_property('contact_number', 'label', 
            frm.doc.membership_type === 'Corporate' ? 'Office Number' : 'Contact Number');
        
        // Field Options
        if (frm.doc.membership_type === 'Student') {
            frm.set_df_property('branch', 'options', 
                frappe.query_reports['Student Library Branches']);
        }
        
        // Field Descriptions
        frm.set_df_property('membership_fee', 'description', 
            `Base fee: $${frm.doc.base_fee || 0}. Includes processing charges.`);
    }
});
```

### Custom Field Widgets

**Creating Custom Controls:**
```javascript
frappe.ui.form.ControlSignature = class ControlSignature extends frappe.ui.form.ControlAttach {
    make() {
        super.make();
        this.setup_signature_pad();
    }
    
    setup_signature_pad() {
        // Custom signature pad implementation
        this.$signature_area = $(`<div class="signature-pad-area">
            <canvas width="400" height="200"></canvas>
            <div class="signature-actions">
                <button class="btn btn-secondary btn-sm clear-signature">Clear</button>
                <button class="btn btn-primary btn-sm save-signature">Save</button>
            </div>
        </div>`).appendTo(this.$wrapper);
        
        this.signature_pad = new SignaturePad(this.$signature_area.find('canvas')[0]);
        
        // Event handlers
        this.$signature_area.find('.clear-signature').click(() => {
            this.signature_pad.clear();
        });
        
        this.$signature_area.find('.save-signature').click(() => {
            this.save_signature();
        });
    }
    
    save_signature() {
        if (this.signature_pad.isEmpty()) {
            frappe.msgprint(__('Please provide a signature'));
            return;
        }
        
        const dataURL = this.signature_pad.toDataURL();
        
        // Upload signature as file
        frappe.call({
            method: 'frappe.utils.file_manager.save_file',
            args: {
                filename: `signature_${this.frm.doc.name}.png`,
                filecontent: dataURL,
                dt: this.frm.doc.doctype,
                dn: this.frm.doc.name
            },
            callback: (r) => {
                if (r.message) {
                    this.set_model_value(r.message.file_url);
                    frappe.msgprint(__('Signature saved successfully'));
                }
            }
        });
    }
};
```

### Advanced Field Formatting

```javascript
frappe.ui.form.on('Library Member', {
    refresh: function(frm) {
        // Custom field formatting
        if (frm.doc.membership_expiry) {
            const days_to_expiry = frappe.datetime.get_diff(frm.doc.membership_expiry, frappe.datetime.get_today());
            
            if (days_to_expiry < 30) {
                frm.dashboard.set_headline_alert(
                    `Membership expires in ${days_to_expiry} days`,
                    'orange'
                );
            }
        }
        
        // Format currency fields
        frm.format_currency('membership_fee', 'USD');
        frm.format_currency('outstanding_balance', 'USD');
        
        // Custom indicators
        const status_color = {
            'Active': 'green',
            'Suspended': 'red', 
            'Expired': 'grey',
            'Pending': 'orange'
        };
        
        if (frm.doc.status) {
            frm.dashboard.add_indicator(__('Status: {0}', [frm.doc.status]), 
                status_color[frm.doc.status] || 'blue');
        }
    }
});
```

## List View Customization

### List View Scripts

**Custom List Behavior:**
```javascript
// apps/library_management/library_management/doctype/library_member/library_member_list.js
frappe.listview_settings['Library Member'] = {
    // Add custom buttons
    add_fields: ["membership_type", "status", "membership_expiry", "outstanding_balance"],
    
    get_indicator: function(doc) {
        // Custom status indicators
        if (doc.status === "Active") {
            if (doc.outstanding_balance > 0) {
                return [__("Outstanding Balance"), "orange", "outstanding_balance,>,0"];
            }
            return [__("Active"), "green", "status,=,Active"];
        } else if (doc.status === "Suspended") {
            return [__("Suspended"), "red", "status,=,Suspended"];
        } else if (doc.status === "Expired") {
            return [__("Expired"), "grey", "status,=,Expired"];
        }
        return [__("Unknown"), "darkgrey", ""];
    },
    
    button: {
        show: function(doc) {
            return doc.status !== "Suspended";
        },
        get_label: function() {
            return __('Generate Report');
        },
        get_description: function(doc) {
            return __('Generate member activity report for {0}', [doc.member_name]);
        },
        action: function(doc) {
            frappe.set_route('query-report', 'Member Activity Report', {member: doc.name});
        }
    },
    
    // Custom formatting
    formatters: {
        member_name: function(value, field, doc) {
            return `<strong>${value}</strong>`;
        },
        
        membership_expiry: function(value, field, doc) {
            if (!value) return '';
            
            const days_diff = frappe.datetime.get_diff(value, frappe.datetime.get_today());
            let color = 'inherit';
            
            if (days_diff < 0) {
                color = '#d73527'; // Expired - red
            } else if (days_diff < 30) {
                color = '#ff8c00'; // Expiring soon - orange  
            }
            
            return `<span style="color: ${color}">${frappe.datetime.str_to_user(value)}</span>`;
        },
        
        outstanding_balance: function(value, field, doc) {
            if (value > 0) {
                return `<span style="color: #d73527">$${frappe.format(value, {fieldtype: 'Currency'})}</span>`;
            }
            return frappe.format(value, {fieldtype: 'Currency'});
        }
    },
    
    // Refresh event
    refresh: function(listview) {
        // Add custom filters
        listview.page.add_menu_item(__("Show Expiring Memberships"), function() {
            const thirty_days_from_now = frappe.datetime.add_days(frappe.datetime.get_today(), 30);
            listview.filter_area.add([
                [listview.doctype, 'membership_expiry', '<=', thirty_days_from_now],
                [listview.doctype, 'status', '=', 'Active']
            ]);
        });
        
        listview.page.add_menu_item(__("Export Member Data"), function() {
            // Custom export functionality
            frappe.call({
                method: 'library_management.api.export_member_data',
                args: {
                    filters: listview.get_filters_for_args()
                },
                callback: function(r) {
                    if (r.message) {
                        window.open(r.message.file_url);
                    }
                }
            });
        });
    },
    
    // Custom actions
    onload: function(listview) {
        // Bulk actions
        listview.page.add_action_item(__("Send Renewal Notices"), function() {
            const selected_items = listview.get_checked_items();
            
            if (selected_items.length === 0) {
                frappe.msgprint(__('Please select members to send notices to'));
                return;
            }
            
            frappe.confirm(__('Send renewal notices to {0} selected members?', [selected_items.length]), 
                function() {
                    frappe.call({
                        method: 'library_management.api.send_renewal_notices',
                        args: {
                            members: selected_items.map(item => item.name)
                        },
                        callback: function(r) {
                            if (r.message) {
                                frappe.msgprint(__('Renewal notices sent successfully'));
                                listview.refresh();
                            }
                        }
                    });
                }
            );
        });
    }
};

// Custom filters
frappe.list_filters_added = function(listview) {
    if (listview.doctype === 'Library Member') {
        // Quick filters
        listview.filter_area.add_quick_filter(__('Active Members'), 'status', '=', 'Active');
        listview.filter_area.add_quick_filter(__('Premium Members'), 'membership_type', '=', 'Premium');
        listview.filter_area.add_quick_filter(__('Expiring Soon'), 'membership_expiry', '<=', 
            frappe.datetime.add_days(frappe.datetime.get_today(), 30));
    }
};
```

## Report Development

### Query Reports

**JavaScript Query Report:**
```javascript
// apps/library_management/library_management/report/member_activity_report/member_activity_report.js
frappe.query_reports["Member Activity Report"] = {
    "filters": [
        {
            "fieldname": "member",
            "label": __("Member"),
            "fieldtype": "Link",
            "options": "Library Member",
            "reqd": 0
        },
        {
            "fieldname": "from_date",
            "label": __("From Date"),
            "fieldtype": "Date",
            "default": frappe.datetime.add_months(frappe.datetime.get_today(), -1),
            "reqd": 1
        },
        {
            "fieldname": "to_date",
            "label": __("To Date"),
            "fieldtype": "Date",
            "default": frappe.datetime.get_today(),
            "reqd": 1
        },
        {
            "fieldname": "status",
            "label": __("Status"),
            "fieldtype": "Select",
            "options": "\nIssued\nReturned\nOverdue",
            "default": ""
        }
    ],
    
    "formatter": function(value, row, column, data, default_formatter) {
        value = default_formatter(value, row, column, data);
        
        // Custom formatting
        if (column.fieldname === "status") {
            if (value === "Overdue") {
                value = `<span style="color: red">${value}</span>`;
            } else if (value === "Returned") {
                value = `<span style="color: green">${value}</span>`;
            }
        }
        
        if (column.fieldname === "fine_amount" && data.fine_amount > 0) {
            value = `<span style="color: red; font-weight: bold">${value}</span>`;
        }
        
        return value;
    },
    
    "onload": function(report) {
        // Add custom buttons
        report.page.add_inner_button(__("Export to Excel"), function() {
            frappe.utils.to_excel(report.data, report.columns, __("Member Activity Report"));
        });
        
        report.page.add_inner_button(__("Send Email"), function() {
            send_report_email(report);
        });
        
        // Set up auto-refresh
        report.auto_refresh_interval = 30000; // 30 seconds
    },
    
    "after_datatable_render": function(datatable_obj) {
        // Custom styling after table renders
        const data = datatable_obj.datamanager.data;
        
        data.forEach((row, index) => {
            if (row[3] === 'Overdue') { // Assuming status is column 3
                datatable_obj.style.setStyle(`.dt-cell[data-row-index="${index}"]`, {
                    backgroundColor: '#fff2f2'
                });
            }
        });
    },
    
    "get_datatable_options": function(options) {
        // Customize datatable options
        return Object.assign(options, {
            checkboxColumn: true,
            events: {
                onCheckRow: function(row) {
                    console.log('Row checked:', row);
                }
            }
        });
    }
};

function send_report_email(report) {
    const dialog = new frappe.ui.Dialog({
        title: __('Email Report'),
        fields: [
            {
                'fieldname': 'recipients',
                'fieldtype': 'Small Text',
                'label': __('Recipients'),
                'reqd': 1,
                'description': __('Comma separated email addresses')
            },
            {
                'fieldname': 'subject',
                'fieldtype': 'Data',
                'label': __('Subject'),
                'default': __('Member Activity Report')
            },
            {
                'fieldname': 'message',
                'fieldtype': 'Text Editor',
                'label': __('Message'),
                'default': __('Please find the attached Member Activity Report.')
            }
        ],
        primary_action_label: __('Send'),
        primary_action: function(values) {
            frappe.call({
                method: 'library_management.api.email_report',
                args: {
                    report_name: 'Member Activity Report',
                    filters: report.get_filters(),
                    recipients: values.recipients,
                    subject: values.subject,
                    message: values.message
                },
                callback: function(r) {
                    if (r.message) {
                        frappe.msgprint(__('Report emailed successfully'));
                        dialog.hide();
                    }
                }
            });
        }
    });
    
    dialog.show();
}
```

### Script Reports

**Python + JavaScript Script Report:**
```javascript
// For script reports (Python-generated)
frappe.query_reports["Library Statistics"] = {
    "filters": [
        {
            "fieldname": "year",
            "label": __("Year"),
            "fieldtype": "Int", 
            "default": new Date().getFullYear()
        }
    ],
    
    "formatter": function(value, row, column, data, default_formatter) {
        // Format based on column type
        if (column.fieldtype === "Currency") {
            return frappe.format(value, column);
        }
        
        if (column.fieldname === "growth_rate") {
            const color = value > 0 ? 'green' : 'red';
            const icon = value > 0 ? '↗' : '↘';
            return `<span style="color: ${color}">${icon} ${value}%</span>`;
        }
        
        return default_formatter(value, row, column, data);
    },
    
    "tree": false,
    "name_field": "month",
    "parent_field": "",
    "initial_depth": 0
};
```

## Dashboard & Workspace Creation

### Custom Dashboards

**Dashboard Configuration:**
```javascript
// Creating custom dashboard widgets
frappe.provide('frappe.dashboards');

frappe.dashboards.library_dashboard = {
    charts: [
        {
            title: "Book Issues This Month",
            chart_type: "line",
            chart_name: "book_issues_trend",
            width: "Half"
        },
        {
            title: "Member Registration",
            chart_type: "bar", 
            chart_name: "member_registration",
            width: "Half"
        }
    ],
    
    shortcuts: [
        {
            label: "New Member",
            link_to: "Library Member",
            type: "DocType"
        },
        {
            label: "Issue Book",
            link_to: "Book Issue", 
            type: "DocType"
        },
        {
            label: "Member Report",
            link_to: "query-report/Member Activity Report",
            type: "Report"
        }
    ]
};

// Custom workspace blocks
frappe.standard_pages['Library Dashboard'] = function() {
    const wrapper = frappe.container.add_page('Library Dashboard');
    
    frappe.ui.make_app_page({
        parent: wrapper,
        title: __('Library Dashboard'),
        single_column: false
    });
    
    // Add custom blocks
    const page = wrapper.page;
    setup_library_dashboard(page);
};

function setup_library_dashboard(page) {
    // Statistics cards
    const stats_section = $(`
        <div class="dashboard-stats-section">
            <div class="row">
                <div class="col-sm-3">
                    <div class="number-card-widget" data-widget="total_members"></div>
                </div>
                <div class="col-sm-3">
                    <div class="number-card-widget" data-widget="books_issued"></div>
                </div>
                <div class="col-sm-3">
                    <div class="number-card-widget" data-widget="overdue_books"></div>
                </div>
                <div class="col-sm-3">
                    <div class="number-card-widget" data-widget="revenue_month"></div>
                </div>
            </div>
        </div>
    `).appendTo(page.main);
    
    // Load dashboard data
    load_dashboard_stats(stats_section);
    
    // Charts section
    const charts_section = $(`
        <div class="dashboard-charts-section">
            <div class="row">
                <div class="col-md-6">
                    <div class="chart-container" id="issues_chart"></div>
                </div>
                <div class="col-md-6"> 
                    <div class="chart-container" id="members_chart"></div>
                </div>
            </div>
        </div>
    `).appendTo(page.main);
    
    load_dashboard_charts(charts_section);
    
    // Quick actions
    setup_quick_actions(page);
}
```

## Dialog & Modal Management

### Custom Dialog Patterns

**Advanced Dialog Usage:**
```javascript
function create_bulk_issue_dialog() {
    const dialog = new frappe.ui.Dialog({
        title: __('Bulk Issue Books'),
        fields: [
            {
                'fieldname': 'member',
                'fieldtype': 'Link',
                'options': 'Library Member',
                'label': __('Member'),
                'reqd': 1,
                'get_query': function() {
                    return {
                        filters: {
                            'status': 'Active'
                        }
                    };
                }
            },
            {
                'fieldname': 'books',
                'fieldtype': 'Table',
                'label': __('Books'),
                'cannot_add_rows': false,
                'cannot_delete_rows': false,
                'fields': [
                    {
                        'fieldname': 'book',
                        'fieldtype': 'Link',
                        'options': 'Library Book',
                        'label': __('Book'),
                        'in_list_view': 1,
                        'reqd': 1,
                        'get_query': function() {
                            return {
                                filters: {
                                    'status': 'Available'
                                }
                            };
                        }
                    },
                    {
                        'fieldname': 'issue_date',
                        'fieldtype': 'Date',
                        'label': __('Issue Date'),
                        'default': frappe.datetime.get_today(),
                        'in_list_view': 1
                    }
                ]
            },
            {
                'fieldname': 'send_notification',
                'fieldtype': 'Check',
                'label': __('Send Email Notification'),
                'default': 1
            }
        ],
        size: 'large',
        primary_action_label: __('Issue Books'),
        primary_action: function(values) {
            if (!values.books || values.books.length === 0) {
                frappe.msgprint(__('Please add at least one book'));
                return;
            }
            
            // Show progress dialog
            const progress_dialog = new frappe.ui.Dialog({
                title: __('Processing Book Issues'),
                fields: [
                    {
                        'fieldname': 'progress',
                        'fieldtype': 'HTML',
                        'options': '<div class="progress"><div class="progress-bar" style="width: 0%"></div></div>'
                    }
                ]
            });
            progress_dialog.show();
            
            // Process issues
            process_bulk_issues(values, progress_dialog).then(() => {
                progress_dialog.hide();
                dialog.hide();
                frappe.msgprint(__('Books issued successfully'));
            });
        }
    });
    
    dialog.show();
    
    // Add custom buttons
    dialog.set_secondary_action_label(__('Import from CSV'));
    dialog.set_secondary_action(function() {
        import_books_from_csv(dialog);
    });
}

async function process_bulk_issues(values, progress_dialog) {
    const books = values.books;
    const total_books = books.length;
    
    for (let i = 0; i < books.length; i++) {
        const book_data = books[i];
        
        try {
            await frappe.call({
                method: 'library_management.api.issue_book',
                args: {
                    member: values.member,
                    book: book_data.book,
                    issue_date: book_data.issue_date
                }
            });
            
            // Update progress
            const progress = ((i + 1) / total_books) * 100;
            progress_dialog.fields_dict.progress.$wrapper.find('.progress-bar').css('width', `${progress}%`);
            
        } catch (error) {
            console.error(`Failed to issue book ${book_data.book}:`, error);
        }
    }
}
```

## Client-Server Communication

### Frappe.call Patterns

**API Communication Examples:**
```javascript
// Basic API call
frappe.call({
    method: 'library_management.api.get_member_dashboard',
    args: {
        member: frm.doc.name
    },
    callback: function(r) {
        if (r.message) {
            // Handle response
            console.log(r.message);
        }
    },
    error: function(r) {
        // Handle error
        frappe.msgprint(__('Failed to load member data'));
    }
});

// Call with freeze (loading indicator)
frappe.call({
    method: 'library_management.reports.generate_monthly_report',
    args: {
        month: '03',
        year: '2024'
    },
    freeze: true,
    freeze_message: __('Generating report...'),
    callback: function(r) {
        if (r.message) {
            window.open(r.message.report_url);
        }
    }
});

// Async/await pattern
async function load_member_books(member_id) {
    try {
        const response = await frappe.call({
            method: 'library_management.api.get_member_books',
            args: { member: member_id }
        });
        
        return response.message || [];
    } catch (error) {
        frappe.msgprint(__('Failed to load member books'));
        return [];
    }
}

// Usage with progress tracking
function bulk_update_members(member_list) {
    const total_members = member_list.length;
    let processed_count = 0;
    
    // Process in batches
    const batch_size = 10;
    const batches = [];
    
    for (let i = 0; i < member_list.length; i += batch_size) {
        batches.push(member_list.slice(i, i + batch_size));
    }
    
    // Process each batch
    batches.forEach((batch, batch_index) => {
        frappe.call({
            method: 'library_management.api.bulk_update_members',
            args: {
                members_data: batch
            },
            callback: function(r) {
                processed_count += batch.length;
                
                // Update progress
                const progress = (processed_count / total_members) * 100;
                frappe.show_progress(__('Updating Members'), progress, 100, 
                    `${processed_count}/${total_members} processed`);
                
                // Hide progress when complete
                if (processed_count >= total_members) {
                    frappe.hide_progress();
                    frappe.msgprint(__('All members updated successfully'));
                }
            }
        });
    });
}
```

### Real-time Updates

```javascript
// Socket.io real-time updates
frappe.realtime.on('book_issued', function(data) {
    // Update UI when book is issued
    if (cur_frm && cur_frm.doctype === 'Library Member' && cur_frm.doc.name === data.member) {
        cur_frm.reload_doc();
    }
    
    // Show notification
    frappe.show_alert({
        message: __('Book "{0}" issued to {1}', [data.book_title, data.member_name]),
        indicator: 'green'
    });
});

frappe.realtime.on('book_returned', function(data) {
    // Handle book return events
    if (cur_list && cur_list.doctype === 'Book Issue') {
        cur_list.refresh();
    }
});

// Send real-time updates from server
function notify_book_issue(member, book) {
    frappe.publish_realtime(
        'book_issued',
        {
            member: member,
            member_name: frappe.get_value('Library Member', member, 'member_name'),
            book: book,
            book_title: frappe.get_value('Library Book', book, 'title')
        }
    );
}
```

---

**Next Steps**: Continue with [API & Integrations](05-api-and-integrations.md) to learn about REST API development, authentication, and third-party integration patterns.