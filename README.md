# ServiceNow_Request_to_Incident_conversion

### Steps to convert Requested Item to Incident (RITM - INC)

1. Create UI Action :*sys_ui_action.DO*
  - Name: Convert to Incident
  - Table: Requested Item [sc_req_item]
  - Show Inset: True
  - Show Update: True
  - Form Button: True
  - Form Context Menu: True
  - Form Style: Destructive
  - List Style: Primary
  - Hint: Closes RITM and Creates Incident
2. Condition: *(current.active == true) && gs.hasRole("itil")*
3. Script
  ```
  /* Construct RITM Links
-----------------------------------------------------------------------------------------------------------------------*/

var ritmTable = current.getTableName();
var ritmSysID = current.getUniqueValue();
var ritmNum = current.getValue('number');

//SERVICE PORTAL: construct a link to the original RITM for inclusion in the customer comments of the incident.
var spRITMLink = spLink(ritmTable,ritmSysID,ritmNum);

//PLATFORM: construct a link to the parent RITM that itil users can use to navigate from the newly created incident.
var platformRITMLink = platformLink(ritmTable,ritmSysID,ritmNum);

/* Construct comments and work notes
-----------------------------------------------------------------------------------------------------------------------*/
var comments = '[code]Incident created from ' + spRITMLink + '<br><br>';

comments += '<strong>Catalog Item Information:</strong><br>';
comments += '<table>';

//Query RITM variables, utilising the variable label and displayValue to build a table.
var vars = current.variables.getElements();
for (var i = 0; i < vars.length; i++){
	var question = vars[i].getQuestion();
	comments += '<tr><td>' + question.getLabel() + ':</td><td>' + question.getDisplayValue() + '</td></tr>';
}
comments += '</table>[/code]';

//include customer comments from the RITM (-1 returns all journal entries for specified journal field)
var ritmComments = current.comments.getJournalEntry(-1);

if (JSUtil.notNil(ritmComments)) {

	comments += '[code]<br><br><strong>Customer comments from ' + ritmNum + '</strong><br>[/code]';
	comments += ritmComments;
}

var workNotes = 'Link to [code]' + platformRITMLink + '[/code] from within ServiceNow.';

/* Ascertain the Assignment Group
-----------------------------------------------------------------------------------------------------------------------*/
var assignmentGroup = getAssignmentGroup(ritmSysID);

/* Create Incident
-----------------------------------------------------------------------------------------------------------------------*/
var inc = new GlideRecord("incident");

inc.caller_id = current.request.requested_for;
inc.assignment_group = assignmentGroup;
inc.short_description = current.short_description;
inc.cmdb_ci = current.cmdb_ci;
inc.impact = current.impact;
inc.urgency = current.urgency;
inc.priority = current.priority;
inc.company = current.company;
inc.sys_domain = current.sys_domain;
inc.comments = comments;
inc.work_notes = workNotes;
inc.parent = ritmSysID;

var incSysID = inc.insert();

var incTable = inc.getTableName();
var incNum = inc.getValue('number');

/* Copy attachments to Incident
-----------------------------------------------------------------------------------------------------------------------*/

//copy any RITM attachments to the newly created incident
GlideSysAttachment.copy(ritmTable, ritmSysID, incTable, incSysID);

/* Construct Incident Links
-----------------------------------------------------------------------------------------------------------------------*/

//SERVICE PORTAL: construct a link to the Incident for inclusion in the customer comments of the RITM.
var spIncLink = spLink(incTable,incSysID,incNum);

//PLATFORM: construct a link to the Incident that itil users can use to navigate from the parent RITM in platform.
var platformIncLink = platformLink(incTable,incSysID,incNum);

/* Update comments and work notes
-----------------------------------------------------------------------------------------------------------------------*/
current.comments = 'RITM converted to [code]' + spIncLink + '[/code]. Please use this reference from now on.';
current.work_notes = 'Link to [code]' + platformIncLink + '[/code] from within ServiceNow.';

/* Close RITM as "Closed Incomplete"
-----------------------------------------------------------------------------------------------------------------------*/
current.setValue('state', '4');
current.setValue('stage', 'Request Cancelled');
current.setValue('active', 'false');

var mySysID = current.update();

gs.addInfoMessage(gs.getMessage("Incident {0} created", incNum));

action.setRedirectURL(inc);
action.setReturnURL(current);

/* Link builders
-----------------------------------------------------------------------------------------------------------------------*/
function spLink(table,sysID,num){

	var link = '<a title="Service Portal Link" href="?id=ticket&table=' + table + '&sys_id=' + sysID + '">' + num + '</a>';

	return link;
}

function platformLink(table,sysID,num){

	var link = '<a title="Platform Link" href="' + table + '.do?sys_id=' + sysID + '">' + num + '</a>';

	return link;
}

/* Locate the last assignment group
-----------------------------------------------------------------------------------------------------------------------*/
function getAssignmentGroup(ritmSysID) {
	/* Return all active tasks relating to RITM ordered by newest task first i.e. last created ticket. */

	var tskGrp = new GlideRecord('sc_task');
	tskGrp.addActiveQuery();
	tskGrp.addQuery('request_item', ritmSysID);
	tskGrp.orderByDesc('number');
	tskGrp.orderByDesc('sys_created_on');
	tskGrp.query();

	var assignmentGroup = '';

	if(tskGrp.next()) {

		//set the assignment group to the last generated active task's assignment group
		assignmentGroup = tskGrp.getValue('assignment_group');
	}
	else {
		/* Default to the Service Desk if a assignment group cannot be found */
		//NOTE: This is a custom system property, containing the group sys_id for your service desk
		//You will need to create it in sys_poperties if you want to fall back to your service desk

		assignmentGroup = gs.getProperty('sc.service_desk');  
	}
	return assignmentGroup;
}

  ```



