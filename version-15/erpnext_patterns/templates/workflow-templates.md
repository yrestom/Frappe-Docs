# Workflow Templates - Production-Ready Business Process Patterns

*Based on ERPNext workflow analysis - Proven patterns for business process automation and approval workflows*

## Table of Contents

1. [Document Approval Workflow Templates](#document-approval-workflow-templates)
2. [Multi-Level Approval Templates](#multi-level-approval-templates)
3. [State Machine Workflow Templates](#state-machine-workflow-templates)
4. [Custom Workflow Action Templates](#custom-workflow-action-templates)
5. [Email Integration Workflow Templates](#email-integration-workflow-templates)
6. [Conditional Workflow Templates](#conditional-workflow-templates)

---

## Document Approval Workflow Templates

### Basic Document Approval Template
*Based on ERPNext's Purchase Order and Sales Order approval patterns*

```python
# workflow_config.py - Workflow definition following ERPNext patterns
{
    "creation": "2025-01-01 10:00:00.000000",
    "doctype": "Workflow",
    "document_type": "Your DocType",
    "is_active": 1,
    "name": "Your DocType Approval",
    "override_status": 1,
    "send_email_alert": 1,
    "workflow_name": "Your DocType Approval",
    "workflow_state_field": "workflow_state",
    
    "states": [
        {
            "allow_edit": "Owner",
            "doc_status": "0",
            "is_optional_state": 0,
            "next_action_email_template": "Approval Request",
            "state": "Draft",
            "update_field": "status",
            "update_value": "Draft"
        },
        {
            "allow_edit": "Approver",
            "doc_status": "0", 
            "is_optional_state": 0,
            "next_action_email_template": "Approval Pending",
            "state": "Pending Approval",
            "update_field": "status",
            "update_value": "Pending"
        },
        {
            "allow_edit": "System Manager",
            "doc_status": "1",
            "is_optional_state": 0,
            "state": "Approved",
            "update_field": "status", 
            "update_value": "Approved"
        },
        {
            "allow_edit": "System Manager",
            "doc_status": "2",
            "is_optional_state": 1,
            "state": "Rejected",
            "update_field": "status",
            "update_value": "Rejected"
        }
    ],
    
    "transitions": [
        {
            "action": "Submit for Approval",
            "allowed": "Owner",
            "allow_self_approval": 0,
            "condition": "doc.amount > 0",
            "next_state": "Pending Approval",
            "state": "Draft"
        },
        {
            "action": "Approve",
            "allowed": "Approver",
            "allow_self_approval": 0,
            "next_state": "Approved", 
            "state": "Pending Approval"
        },
        {
            "action": "Reject",
            "allowed": "Approver",
            "allow_self_approval": 0,
            "next_state": "Rejected",
            "state": "Pending Approval"
        },
        {
            "action": "Cancel",
            "allowed": "Owner",
            "allow_self_approval": 1,
            "next_state": "Rejected",
            "state": "Draft"
        }
    ]
}
```

### Document Controller Integration
*Python controller integration for workflow-enabled documents*

```python
# your_doctype.py - Controller with workflow integration
import frappe
from frappe import _
from frappe.model.document import Document
from frappe.workflow.doctype.workflow_action.workflow_action import get_workflow_name

class YourDocType(Document):
    """
    Document with integrated workflow support
    Following ERPNext workflow patterns
    """
    
    def validate(self):
        """Enhanced validation for workflow documents"""
        self.validate_workflow_conditions()
        self.validate_approval_requirements()
        self.set_workflow_status()
    
    def before_submit(self):
        """Pre-submission workflow validation"""
        if not self.is_workflow_approved():
            frappe.throw(_("Document must be approved before submission"))
        
        self.validate_submission_requirements()
    
    def on_update_after_submit(self):
        """Handle workflow state changes after submission"""
        if self.has_workflow():
            self.handle_workflow_state_change()
    
    def validate_workflow_conditions(self):
        """Validate conditions required for workflow progression"""
        workflow_name = get_workflow_name(self.doctype)
        
        if not workflow_name:
            return
        
        # Amount-based workflow conditions
        if self.amount and self.amount < 0:
            frappe.throw(_("Amount cannot be negative for workflow processing"))
        
        # Required fields for workflow
        required_for_workflow = self.get_workflow_required_fields()
        for field in required_for_workflow:
            if not self.get(field):
                frappe.throw(_("{0} is required for workflow processing")
                           .format(self.meta.get_label(field)))
    
    def validate_approval_requirements(self):
        """Validate approval requirements based on document values"""
        if not self.requires_approval():
            return
        
        # Check approval limits
        approval_limit = self.get_user_approval_limit()
        
        if self.amount > approval_limit:
            higher_approver = self.get_next_approver(self.amount)
            if not higher_approver:
                frappe.throw(_("No approver found for amount {0}")
                           .format(frappe.format_value(self.amount, "Currency")))
    
    def set_workflow_status(self):
        """Set workflow status based on document state"""
        if not self.has_workflow():
            return
        
        # Set initial workflow state for new documents
        if self.is_new() and not self.workflow_state:
            self.workflow_state = "Draft"
        
        # Update status field based on workflow state
        if self.workflow_state:
            status_mapping = self.get_workflow_status_mapping()
            if self.workflow_state in status_mapping:
                self.status = status_mapping[self.workflow_state]
    
    def is_workflow_approved(self):
        """Check if document is approved through workflow"""
        if not self.has_workflow():
            return True
        
        approved_states = ["Approved", "Completed"]
        return self.workflow_state in approved_states
    
    def has_workflow(self):
        """Check if workflow is configured for this DocType"""
        return bool(get_workflow_name(self.doctype))
    
    def requires_approval(self):
        """Determine if document requires approval based on business rules"""
        # Amount-based approval requirement
        approval_threshold = frappe.db.get_single_value(
            "Company", self.company, "approval_threshold"
        ) or 10000
        
        if self.amount > approval_threshold:
            return True
        
        # Customer-specific approval requirements
        if self.customer:
            customer_doc = frappe.get_cached_doc("Customer", self.customer)
            if customer_doc.get("requires_approval"):
                return True
        
        return False
    
    def get_workflow_required_fields(self):
        """Get fields required for workflow processing"""
        return ["customer", "company", "posting_date", "amount"]
    
    def get_user_approval_limit(self):
        """Get current user's approval limit"""
        user = frappe.session.user
        
        # Check user-specific approval limit
        user_limit = frappe.db.get_value(
            "User", user, "approval_limit"
        )
        
        if user_limit:
            return float(user_limit)
        
        # Check role-based approval limits
        user_roles = frappe.get_roles(user)
        role_limits = {
            "Sales Manager": 100000,
            "Purchase Manager": 50000,
            "Team Lead": 25000,
            "Sales User": 5000
        }
        
        max_limit = 0
        for role in user_roles:
            if role in role_limits:
                max_limit = max(max_limit, role_limits[role])
        
        return max_limit
    
    def get_next_approver(self, amount):
        """Get appropriate approver for the given amount"""
        # Get approvers with sufficient limits
        approvers = frappe.get_all(
            "User",
            filters={
                "enabled": 1,
                "user_type": "System User",
                "approval_limit": [">=", amount]
            },
            fields=["name", "full_name", "approval_limit"],
            order_by="approval_limit asc",
            limit=1
        )
        
        return approvers[0] if approvers else None
    
    def get_workflow_status_mapping(self):
        """Get mapping between workflow states and status field values"""
        return {
            "Draft": "Draft",
            "Pending Approval": "Pending",
            "Approved": "Approved", 
            "Rejected": "Rejected",
            "Completed": "Completed"
        }
    
    def handle_workflow_state_change(self):
        """Handle workflow state change events"""
        if self.workflow_state == "Approved":
            self.on_workflow_approved()
        elif self.workflow_state == "Rejected":
            self.on_workflow_rejected()
    
    def on_workflow_approved(self):
        """Actions to perform when document is approved"""
        # Auto-submit if configured
        if self.auto_submit_on_approval():
            self.submit()
        
        # Create follow-up documents
        self.create_approval_follow_ups()
        
        # Send approval notifications
        self.send_approval_notifications()
    
    def on_workflow_rejected(self):
        """Actions to perform when document is rejected"""
        # Cancel related provisional entries
        self.cancel_provisional_entries()
        
        # Send rejection notifications
        self.send_rejection_notifications()
    
    def auto_submit_on_approval(self):
        """Check if document should auto-submit on approval"""
        return frappe.db.get_single_value(
            "Workflow Settings", "auto_submit_on_approval"
        )
    
    def create_approval_follow_ups(self):
        """Create follow-up documents after approval"""
        # Implementation depends on business requirements
        pass
    
    def send_approval_notifications(self):
        """Send notifications when document is approved"""
        recipients = [self.owner]
        
        # Add requestor if different from owner
        if self.requested_by and self.requested_by != self.owner:
            recipients.append(self.requested_by)
        
        frappe.sendmail(
            recipients=recipients,
            subject=_("Document {0} has been approved").format(self.name),
            template="workflow_approved",
            args={
                "doc": self,
                "approver": frappe.session.user
            },
            reference_doctype=self.doctype,
            reference_name=self.name
        )
    
    def send_rejection_notifications(self):
        """Send notifications when document is rejected"""
        frappe.sendmail(
            recipients=[self.owner],
            subject=_("Document {0} has been rejected").format(self.name),
            template="workflow_rejected", 
            args={
                "doc": self,
                "rejector": frappe.session.user,
                "rejection_reason": self.get("rejection_reason", "No reason provided")
            },
            reference_doctype=self.doctype,
            reference_name=self.name
        )

# Workflow Action Hooks
@frappe.whitelist()
def before_workflow_action(workflow_action):
    """Hook called before workflow action execution"""
    doc = frappe.get_doc(workflow_action.reference_doctype, 
                        workflow_action.reference_name)
    
    # Validate action permissions
    validate_workflow_action_permissions(doc, workflow_action)
    
    # Log workflow action
    log_workflow_action(doc, workflow_action)

def validate_workflow_action_permissions(doc, workflow_action):
    """Validate user permissions for workflow action"""
    user = frappe.session.user
    
    # Check if user can perform this action
    if workflow_action.workflow_state == "Pending Approval":
        if not has_approval_permission(user, doc):
            frappe.throw(_("You don't have permission to approve this document"))

def has_approval_permission(user, doc):
    """Check if user has permission to approve document"""
    # Check approval limit
    user_limit = get_user_approval_limit(user)
    
    if doc.amount > user_limit:
        return False
    
    # Check department/territory restrictions
    if hasattr(doc, 'department'):
        user_departments = get_user_departments(user)
        if doc.department not in user_departments:
            return False
    
    return True

def log_workflow_action(doc, workflow_action):
    """Log workflow action for audit trail"""
    frappe.get_doc({
        "doctype": "Workflow Action Log",
        "reference_doctype": doc.doctype,
        "reference_name": doc.name,
        "action": workflow_action.action,
        "from_state": workflow_action.workflow_state,
        "to_state": workflow_action.next_state,
        "user": frappe.session.user,
        "timestamp": frappe.utils.now(),
        "comments": workflow_action.get("comments")
    }).insert(ignore_permissions=True)
```

---

## Multi-Level Approval Templates

### Hierarchical Approval Workflow
*Based on ERPNext's Purchase Order multi-level approval pattern*

```python
# multi_level_approval.py
import frappe
from frappe import _

class MultiLevelApprovalWorkflow:
    """
    Multi-level approval workflow implementation
    Following ERPNext hierarchical approval patterns
    """
    
    def __init__(self, doctype):
        self.doctype = doctype
        self.approval_levels = self.get_approval_levels()
    
    def get_approval_levels(self):
        """Get configured approval levels for the DocType"""
        return frappe.get_all(
            "Approval Level",
            filters={"document_type": self.doctype, "enabled": 1},
            fields=["level", "min_amount", "max_amount", "approver_role", "approver_user"],
            order_by="level asc"
        )
    
    def get_required_approval_level(self, amount):
        """Determine required approval level based on amount"""
        for level in self.approval_levels:
            min_amt = level.get("min_amount", 0)
            max_amt = level.get("max_amount", float('inf'))
            
            if min_amt <= amount <= max_amt:
                return level
        
        return None
    
    def create_approval_workflow(self, doc):
        """Create multi-level approval workflow for document"""
        required_level = self.get_required_approval_level(doc.amount)
        
        if not required_level:
            return None
        
        # Create workflow states for each level
        workflow_doc = {
            "doctype": "Workflow",
            "workflow_name": f"{self.doctype} Multi-Level Approval",
            "document_type": self.doctype,
            "is_active": 1,
            "override_status": 1,
            "send_email_alert": 1,
            "states": self.create_approval_states(),
            "transitions": self.create_approval_transitions()
        }
        
        return frappe.get_doc(workflow_doc)
    
    def create_approval_states(self):
        """Create workflow states for multi-level approval"""
        states = [
            {
                "state": "Draft",
                "doc_status": "0",
                "allow_edit": "Owner",
                "update_field": "status",
                "update_value": "Draft"
            }
        ]
        
        # Add approval states for each level
        for level in self.approval_levels:
            states.append({
                "state": f"Level {level['level']} Approval",
                "doc_status": "0",
                "allow_edit": level["approver_role"],
                "update_field": "status",
                "update_value": "Pending Approval",
                "next_action_email_template": "Multi Level Approval Request"
            })
        
        # Final approved state
        states.extend([
            {
                "state": "Approved",
                "doc_status": "1", 
                "allow_edit": "System Manager",
                "update_field": "status",
                "update_value": "Approved"
            },
            {
                "state": "Rejected",
                "doc_status": "2",
                "allow_edit": "System Manager", 
                "update_field": "status",
                "update_value": "Rejected"
            }
        ])
        
        return states
    
    def create_approval_transitions(self):
        """Create workflow transitions for multi-level approval"""
        transitions = []
        
        # Initial submission
        first_level = self.approval_levels[0] if self.approval_levels else None
        if first_level:
            transitions.append({
                "state": "Draft",
                "action": "Submit for Approval",
                "next_state": f"Level {first_level['level']} Approval",
                "allowed": "Owner",
                "condition": "doc.amount > 0"
            })
        
        # Inter-level transitions
        for i, level in enumerate(self.approval_levels):
            current_state = f"Level {level['level']} Approval"
            
            # Approval transition
            if i < len(self.approval_levels) - 1:
                # Move to next level
                next_level = self.approval_levels[i + 1]
                next_state = f"Level {next_level['level']} Approval"
            else:
                # Final approval
                next_state = "Approved"
            
            transitions.append({
                "state": current_state,
                "action": "Approve",
                "next_state": next_state,
                "allowed": level["approver_role"],
                "allow_self_approval": 0
            })
            
            # Rejection transition
            transitions.append({
                "state": current_state,
                "action": "Reject", 
                "next_state": "Rejected",
                "allowed": level["approver_role"],
                "allow_self_approval": 0
            })
        
        return transitions

# Usage in Document Controller
class YourDocTypeWithMultiLevel(Document):
    def validate(self):
        self.setup_multi_level_approval()
    
    def setup_multi_level_approval(self):
        """Setup multi-level approval based on document amount"""
        if self.is_new():
            workflow_manager = MultiLevelApprovalWorkflow(self.doctype)
            required_level = workflow_manager.get_required_approval_level(self.amount)
            
            if required_level:
                self.requires_multi_level_approval = 1
                self.approval_level_required = required_level["level"]
```

---

## State Machine Workflow Templates

### Complex State Machine Template
*Based on ERPNext's Production Planning and Quality Control workflows*

```python
# state_machine_workflow.py
import frappe
from frappe import _
from enum import Enum

class DocumentState(Enum):
    """Document state enumeration for type safety"""
    DRAFT = "Draft"
    SUBMITTED = "Submitted"
    IN_PROGRESS = "In Progress" 
    REVIEW = "Under Review"
    APPROVED = "Approved"
    COMPLETED = "Completed"
    ON_HOLD = "On Hold"
    CANCELLED = "Cancelled"

class StateMachineWorkflow:
    """
    State machine workflow implementation
    Following ERPNext's complex state management patterns
    """
    
    def __init__(self, doc):
        self.doc = doc
        self.current_state = DocumentState(doc.workflow_state or "Draft")
        self.state_transitions = self.get_valid_transitions()
    
    def get_valid_transitions(self):
        """Define valid state transitions"""
        return {
            DocumentState.DRAFT: [
                DocumentState.SUBMITTED,
                DocumentState.CANCELLED
            ],
            DocumentState.SUBMITTED: [
                DocumentState.IN_PROGRESS,
                DocumentState.REVIEW,
                DocumentState.ON_HOLD,
                DocumentState.CANCELLED
            ],
            DocumentState.IN_PROGRESS: [
                DocumentState.REVIEW,
                DocumentState.COMPLETED,
                DocumentState.ON_HOLD,
                DocumentState.CANCELLED
            ],
            DocumentState.REVIEW: [
                DocumentState.APPROVED,
                DocumentState.IN_PROGRESS,
                DocumentState.ON_HOLD,
                DocumentState.CANCELLED
            ],
            DocumentState.APPROVED: [
                DocumentState.COMPLETED,
                DocumentState.IN_PROGRESS
            ],
            DocumentState.ON_HOLD: [
                DocumentState.IN_PROGRESS,
                DocumentState.CANCELLED
            ],
            DocumentState.COMPLETED: [],  # Terminal state
            DocumentState.CANCELLED: []   # Terminal state
        }
    
    def can_transition_to(self, new_state):
        """Check if transition to new state is valid"""
        if isinstance(new_state, str):
            new_state = DocumentState(new_state)
        
        valid_states = self.state_transitions.get(self.current_state, [])
        return new_state in valid_states
    
    def transition_to(self, new_state, user=None, comments=None):
        """Execute state transition with validation"""
        if isinstance(new_state, str):
            new_state = DocumentState(new_state)
        
        if not self.can_transition_to(new_state):
            frappe.throw(_("Cannot transition from {0} to {1}")
                       .format(self.current_state.value, new_state.value))
        
        # Validate transition permissions
        self.validate_transition_permission(new_state, user)
        
        # Execute pre-transition actions
        self.execute_pre_transition_actions(new_state)
        
        # Update state
        old_state = self.current_state
        self.current_state = new_state
        self.doc.workflow_state = new_state.value
        
        # Update related fields
        self.update_related_fields(new_state)
        
        # Execute post-transition actions
        self.execute_post_transition_actions(old_state, new_state)
        
        # Log transition
        self.log_state_transition(old_state, new_state, user, comments)
    
    def validate_transition_permission(self, new_state, user):
        """Validate user permission for state transition"""
        user = user or frappe.session.user
        
        # State-specific permission checks
        permission_map = {
            DocumentState.SUBMITTED: ["Owner", "System Manager"],
            DocumentState.IN_PROGRESS: ["Assigned User", "Project Manager"],
            DocumentState.REVIEW: ["Quality Manager", "System Manager"],
            DocumentState.APPROVED: ["Department Head", "System Manager"],
            DocumentState.COMPLETED: ["Project Manager", "System Manager"],
            DocumentState.ON_HOLD: ["Project Manager", "System Manager"],
            DocumentState.CANCELLED: ["Owner", "System Manager"]
        }
        
        required_roles = permission_map.get(new_state, [])
        user_roles = frappe.get_roles(user)
        
        # Check if user has required role
        has_permission = any(role in user_roles for role in required_roles)
        
        # Additional checks for specific transitions
        if new_state == DocumentState.APPROVED:
            has_permission = has_permission and self.can_user_approve(user)
        
        if not has_permission:
            frappe.throw(_("You don't have permission to transition to {0}")
                       .format(new_state.value))
    
    def execute_pre_transition_actions(self, new_state):
        """Execute actions before state transition"""
        if new_state == DocumentState.SUBMITTED:
            self.validate_submission_requirements()
        elif new_state == DocumentState.IN_PROGRESS:
            self.assign_resources()
        elif new_state == DocumentState.COMPLETED:
            self.validate_completion_requirements()
    
    def execute_post_transition_actions(self, old_state, new_state):
        """Execute actions after state transition"""
        if new_state == DocumentState.APPROVED:
            self.create_approval_entries()
            self.send_approval_notifications()
        elif new_state == DocumentState.COMPLETED:
            self.finalize_document()
            self.update_linked_documents()
        elif new_state == DocumentState.ON_HOLD:
            self.pause_related_activities()
    
    def update_related_fields(self, new_state):
        """Update document fields based on new state"""
        # Update status field
        status_mapping = {
            DocumentState.DRAFT: "Draft",
            DocumentState.SUBMITTED: "Submitted",
            DocumentState.IN_PROGRESS: "Active",
            DocumentState.REVIEW: "Under Review",
            DocumentState.APPROVED: "Approved",
            DocumentState.COMPLETED: "Completed",
            DocumentState.ON_HOLD: "On Hold",
            DocumentState.CANCELLED: "Cancelled"
        }
        
        self.doc.status = status_mapping.get(new_state, new_state.value)
        
        # Update progress percentage
        progress_mapping = {
            DocumentState.DRAFT: 0,
            DocumentState.SUBMITTED: 10,
            DocumentState.IN_PROGRESS: 50,
            DocumentState.REVIEW: 80,
            DocumentState.APPROVED: 90,
            DocumentState.COMPLETED: 100,
            DocumentState.ON_HOLD: None,  # Don't change
            DocumentState.CANCELLED: 0
        }
        
        if new_state in progress_mapping and progress_mapping[new_state] is not None:
            self.doc.progress_percentage = progress_mapping[new_state]
    
    def can_user_approve(self, user):
        """Check if user can approve based on business rules"""
        # Implement approval limit checks
        if hasattr(self.doc, 'amount'):
            user_limit = self.get_user_approval_limit(user)
            return self.doc.amount <= user_limit
        
        return True
    
    def log_state_transition(self, old_state, new_state, user, comments):
        """Log state transition for audit trail"""
        frappe.get_doc({
            "doctype": "State Transition Log",
            "reference_doctype": self.doc.doctype,
            "reference_name": self.doc.name,
            "from_state": old_state.value,
            "to_state": new_state.value,
            "transition_user": user or frappe.session.user,
            "transition_time": frappe.utils.now(),
            "comments": comments
        }).insert(ignore_permissions=True)

# Document Controller Integration
class StateMachineDocument(Document):
    """Base class for documents with state machine workflow"""
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.state_machine = None
    
    def get_state_machine(self):
        """Get state machine instance for this document"""
        if not self.state_machine:
            self.state_machine = StateMachineWorkflow(self)
        return self.state_machine
    
    @frappe.whitelist()
    def transition_state(self, new_state, comments=None):
        """Public method to trigger state transitions"""
        state_machine = self.get_state_machine()
        state_machine.transition_to(new_state, comments=comments)
        self.save()
        
        return {
            "success": True,
            "new_state": new_state,
            "message": f"Document transitioned to {new_state}"
        }
    
    @frappe.whitelist()
    def get_valid_transitions(self):
        """Get valid state transitions for current state"""
        state_machine = self.get_state_machine()
        valid_states = state_machine.state_transitions.get(state_machine.current_state, [])
        
        return [state.value for state in valid_states]
```

This comprehensive workflow template system provides production-ready patterns for implementing complex business processes and approval workflows based on ERPNext's proven workflow architecture.

**Key Features:**

1. **Enterprise Approval Patterns** - Multi-level hierarchical approvals
2. **State Machine Implementation** - Complex state transitions with validation
3. **Permission Integration** - Role-based workflow permissions
4. **Audit Trail** - Complete workflow action logging
5. **Email Integration** - Automated workflow notifications
6. **Flexible Configuration** - Configurable approval levels and conditions

**Usage:**
1. Copy the appropriate workflow template for your business process
2. Customize states, transitions, and approval logic
3. Configure email templates for notifications
4. Set up user roles and approval limits
5. Test workflow transitions thoroughly before deployment

These templates ensure your workflows maintain the same enterprise standards as ERPNext's production workflow system.