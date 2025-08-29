# Privileged Identity Management (PIM) - Getting Started

!!! info "Enterprise Zero-Trust Security"
    This guide covers implementing Microsoft Entra Privileged Identity Management (PIM) for enterprise environments, focusing on zero-trust principles and just-in-time access management.

## Overview

Privileged Identity Management (PIM) is a service in Microsoft Entra ID that enables you to manage, control, and monitor access to important resources in your organisation. PIM provides time-based and approval-based role activation to mitigate the risks of excessive, unnecessary, or misused access permissions.

### Key Benefits

=== "Security Enhancement"

    **Zero Standing Privileges**

    - Eliminates permanent administrative access
    - Reduces attack surface through just-in-time access
    - Provides comprehensive audit trails for all privileged operations
    - Implements approval workflows for sensitive role activations

=== "Compliance & Governance"

    **Regulatory Compliance**

    - Meets SOX, PCI-DSS, and other regulatory requirements
    - Provides detailed access reviews and reporting
    - Implements segregation of duties principles
    - Maintains comprehensive audit logs for compliance reporting

=== "Operational Efficiency"

    **Streamlined Administration**

    - Automates access provisioning and deprovisioning
    - Reduces administrative overhead through self-service activation
    - Provides centralised management of privileged access
    - Integrates with existing identity governance processes

## Prerequisites

### Licensing Requirements

!!! warning "Licensing Dependency"
    PIM requires **Microsoft Entra ID P2** or **Microsoft 365 E5** licensing for all users who will have privileged roles managed by PIM.

- **Microsoft Entra ID P2**: Standalone licensing option
- **Microsoft 365 E5**: Includes Entra ID P2 capabilities
- **Enterprise Mobility + Security E5**: Alternative licensing path

### Required Permissions

To configure PIM, you need one of the following roles:

- **Global Administrator**: Full access to all PIM features
- **Privileged Role Administrator**: Can manage role assignments and settings
- **Security Administrator**: Can view PIM data and configure some settings

### Technical Prerequisites

- **Microsoft Entra ID tenant** with appropriate licensing
- **Multi-Factor Authentication (MFA)** configured for privileged users
- **Conditional Access policies** for enhanced security
- **Azure AD Connect** (if using hybrid identity)

## Initial Setup and Configuration

### Step 1: Enable PIM

1. **Navigate to Microsoft Entra Admin Center**
   ```
   https://entra.microsoft.com
   ```

2. **Access PIM Service**
    - Go to **Identity governance** > **Privileged Identity Management**
    - Click **Consent to PIM** if prompted
    - Review and accept the terms of service

3. **Verify Service Activation**
    - Confirm PIM dashboard is accessible
    - Verify licensing compliance warnings (if any)

### Step 2: Configure Role Settings

=== "Microsoft Entra Roles"

    **Configure Built-in Roles**

    1. Navigate to **Microsoft Entra roles** > **Settings**
    2. Select a role to configure (start with Global Administrator)
    3. Configure activation settings:
       - **Maximum activation duration**: 4-8 hours recommended
       - **Require approval**: Enable for high-privilege roles
       - **Require MFA**: Always enable
       - **Require justification**: Enable for audit purposes

=== "Azure Resource Roles"

    **Configure Subscription/Resource Roles**

    1. Navigate to **Azure resources** > **Discover resources**
    2. Select subscriptions or management groups to manage
    3. Configure role settings similar to Entra roles
    4. Set up resource-specific approval workflows

### Step 3: Assign Eligible Roles

1. **Remove Permanent Assignments**
    - Audit existing permanent role assignments
    - Convert to eligible assignments where appropriate
    - Maintain emergency access accounts as permanent

2. **Create Eligible Assignments**
   ```
   Navigate to: PIM > Microsoft Entra roles > Assignments
   - Click "Add assignments"
   - Select role and users/groups
   - Choose "Eligible" assignment type
   - Set start/end dates if required
   ```

3. **Configure Assignment Settings**
    - **Assignment duration**: Typically 6-12 months
    - **Require justification**: For assignment creation
    - **Notification settings**: Configure for role assignments

## Role Configuration Best Practices

### High-Privilege Roles Configuration

For roles like Global Administrator, Privileged Role Administrator:

```yaml
Activation Settings:
  Maximum Duration: 4 hours
  Require Approval: Yes
  Require MFA: Yes
  Require Justification: Yes
  Require Ticket Information: Yes (if using ITSM)

Assignment Settings:
  Maximum Duration: 6 months
  Require Approval: Yes
  Require Justification: Yes

Notification Settings:
  Notify on Activation: All admins
  Notify on Assignment: Security team
```

### Medium-Privilege Roles Configuration

For roles like User Administrator, Exchange Administrator:

```yaml
Activation Settings:
  Maximum Duration: 8 hours
  Require Approval: No (with conditions)
  Require MFA: Yes
  Require Justification: Yes

Assignment Settings:
  Maximum Duration: 12 months
  Require Approval: No
  Require Justification: Yes
```

### Approval Workflows

=== "Single Approver"

    **Simple Approval Process**

    - Suitable for medium-privilege roles
    - Single approver from security team
    - Automatic approval after timeout (optional)

=== "Multi-Stage Approval"

    **Complex Approval Process**

    - Required for high-privilege roles
    - Multiple approvers (manager + security)
    - No automatic approval
    - Escalation procedures defined

## Security Hardening

### Multi-Factor Authentication

!!! critical "MFA Requirement"
    Always require MFA for privileged role activation. This is non-negotiable for enterprise security.

**Configuration Steps:**

1. Navigate to **Conditional Access**
2. Create policy for PIM activation
3. Require MFA for all privileged role activations
4. Consider requiring compliant devices

### Conditional Access Integration

Create specific Conditional Access policies for PIM:

```yaml
Policy Name: "PIM Activation - High Privilege Roles"
Assignments:
  Users: All eligible for high-privilege roles
  Cloud Apps: Microsoft Azure Management
  Conditions:
    - Sign-in risk: Medium and High
    - Device platforms: All

Access Controls:
  Grant: Require MFA + Compliant Device
  Session: Sign-in frequency every 4 hours
```

### Emergency Access Procedures

Maintain emergency access accounts:

- **Break-glass accounts**: 2-3 cloud-only accounts
- **Permanent assignments**: Only for emergency accounts
- **Strong passwords**: 25+ character complex passwords
- **Secure storage**: Hardware security modules or secure vaults
- **Regular testing**: Monthly access verification

## Monitoring and Alerting

### Built-in Monitoring

PIM provides comprehensive monitoring capabilities:

1. **Activation History**
    - All role activations with timestamps
    - Justifications and approval chains
    - Duration and scope of access

2. **Assignment Changes**
    - Role assignment modifications
    - Eligibility changes
    - Configuration updates

3. **Access Reviews**
    - Periodic review of role assignments
    - Automated removal of unused assignments
    - Compliance reporting

### Custom Alerting

Configure alerts for:

- **Unusual activation patterns**: Multiple roles activated simultaneously
- **Failed activations**: Repeated failed attempts
- **Emergency account usage**: Any use of break-glass accounts
- **Configuration changes**: Modifications to PIM settings

### Integration with SIEM

Export PIM logs to your Security Information and Event Management (SIEM) system:

```powershell
# Example: Export PIM audit logs
Connect-MgGraph -Scopes "AuditLog.Read.All"
Get-MgAuditLogDirectoryAudit -Filter "category eq 'RoleManagement'" |
    Export-Csv -Path "PIM-Audit-$(Get-Date -Format 'yyyy-MM-dd').csv"
```

## Common Implementation Challenges

### User Adoption

**Challenge**: Users resistant to additional activation steps

**Solution**:

- Comprehensive training programmes
- Clear documentation and self-service guides
- Gradual rollout starting with IT teams

### Role Sprawl

**Challenge**: Too many custom roles created

**Solution**:

- Regular role audits and cleanup
- Standardised role definitions
- Principle of least privilege enforcement

### Approval Bottlenecks

**Challenge**: Approval processes causing delays

**Solution**:

- Multiple approvers for redundancy
- Clear SLA definitions
- Escalation procedures
- Consider auto-approval for lower-risk scenarios

## Next Steps

After implementing basic PIM:

1. **[Advanced Group Management](Groups.md)**: Implement PIM for Groups
2. **Access Reviews**: Set up periodic access reviews
3. **Integration**: Connect with ITSM and SIEM systems
4. **Automation**: Implement PowerShell/Graph API automation
5. **Governance**: Establish ongoing governance processes

## Troubleshooting Common Issues

### Activation Failures

```powershell
# Check user's eligible assignments
Get-MgRoleManagementDirectoryRoleEligibilitySchedule -Filter "principalId eq 'user-object-id'"

# Verify MFA status
Get-MgUser -UserId "user@domain.com" -Property "StrongAuthenticationMethods"
```

### Permission Issues

- Verify licensing compliance
- Check Conditional Access policy impacts
- Validate MFA registration status
- Review role assignment scope

!!! tip "Enterprise Implementation"
    Start with a pilot group of IT administrators before rolling out to the entire organisation. This allows you to refine processes and address issues before full deployment.