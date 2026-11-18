var ContractsAdohocApprovalHelperSNC = Class.create();
ContractsAdohocApprovalHelperSNC.prototype = {
  initialize: function () {
    this.APPROVAL_TABLE = 'sysapproval_approver';
    this.GROUP_APPROVAL_TABLE = 'sysapproval_group';
  },

  /**
   *
   * To get the available Contract Types for the given Contract Request
   * @property {json} input - Contract Request details such as table name, sysid.
   * */

  getContractTypes: function (input) {
    var table = input.table;
    var sysId = input.sysId;
    var contractTypes = [];
    if (gs.nil(table) || gs.nil(sysId)) return {};
    var request = new GlideRecordSecure(table);
    if (request.get(sysId)) {
      if (table == CMConstants.TABLES.ContractRequest.name) {
        var contractDocument =
          new sn_cm_core.ContractDocumentUtils().getDocumentsByTarget(sysId);
        contractTypes = RecordList(contractDocument)
          .map(function (document) {
            if (!gs.nil(document.default_version)) {
              return {
                id: document.getUniqueValue(),
                label: document.getDisplayValue(),
              };
            }
          })
          .filter(document => {
            return document != null && document != undefined;
          });
      }
    }
    return contractTypes;
  },

  /**
   *
   * To get the revision record which is in ready state for a given contract type
   * @property {SysID} contractType - Sys ID of the Contract Type.
   * */

  getDocumentReadyRevision: function (contractType) {
    var revisionGr = new GlideRecord(CMConstants.TABLES.DocumentRevision.name);
    revisionGr.addQuery('document', contractType);
    var readyCondition = revisionGr.addQuery('ready', 1);
    readyCondition.addOrCondition('ready', true);
    revisionGr.orderByDesc('sys_created_on');
    revisionGr.query();
    if (revisionGr.next()) {
      if (this.checkIfApprovalExists(revisionGr)) {
        return {
          id: revisionGr.getUniqueValue(),
          label: revisionGr.getDisplayValue(),
          approvalExists: true,
        };
      } else {
        return {
          id: revisionGr.getUniqueValue(),
          label: revisionGr.getDisplayValue(),
          approvalExists: false,
        };
      }
    } else {
      return null;
    }
  },

  /**
   *
   * To check if the approval exists for a given revision record.
   * @property {GlideRecord} revisionGr - Revision Glide Record
   * */
  checkIfApprovalExists: function (revisionGr) {
    var approval = new GlideRecord(this.APPROVAL_TABLE);
    approval.addQuery('source_table', revisionGr.getTableName());
    approval.addQuery('document_id', revisionGr.getUniqueValue());
    approval.addQuery('state', 'requested');
    approval.query();
    if (approval.next()) {
      return true;
    } else {
      return false;
    }
  },

  /**
   *
   * To create a User Approval for the given details.
   * */
  createUserApproval: function (comments, requestId, userId, documentId) {
    if(!this.isUserHasRequiredRole(userId,'sn_cm_core.contract_fulfiller')){
      throw new Error('Selected User does not have required role');
    }
    var app = new GlideRecord(this.APPROVAL_TABLE);
    app.initialize();
    app.source_table = CMConstants.TABLES.DocumentRevision.name;
    app.document_id = documentId;
    app.comments = comments;
    app.state = 'requested';
    app.approver = userId;
    var approvalSysId = app.insert();
    if (approvalSysId) {
        var gr = new GlideRecord(this.APPROVAL_TABLE);
        if (gr.get(approvalSysId)) {
            gr.sysapproval = requestId;
            gr.update();
        }
    }

    var contractReq = new GlideRecord(CMConstants.TABLES.ContractRequest.name);
    if (contractReq.get(requestId)) {
        this.createPermissionsForAdhocApproval( 
            contractReq.getValue('folder_id'),
            'user',
            userId,
            requestId,
            CMConstants.TABLES.ContractRequest.name,
        );
    }
  },

  isUserHasRequiredRole: function(userSysId, role) {
      var gr = new GlideRecord('sys_user_has_role');
      gr.addQuery('user', userSysId);
      gr.addQuery('role.name', role);
      gr.query();
      if (gr.next()) {
		    return true;
      } else {
		    return false;
      }
  },

  /**
   *
   * To create a Group Approval for the given details.
   * */
  createGroupApproval: function (comments, requestId, groupId, documentId) {
    if(!this.isUserGroupHasRequiredRole(groupId,'sn_cm_core.contract_fulfiller')){
      throw new Error('Selected User Group does not have required role');
    }
    var app = new GlideRecord(this.GROUP_APPROVAL_TABLE);
    app.initialize();
    app.parent = requestId;
    app.approval = 'requested';
    app.assignment_group = groupId;
    var groupRecord = app.insert();
    if (!gs.nil(groupRecord)) {
      var approvalRecord = new GlideRecord(this.APPROVAL_TABLE);
      approvalRecord.addQuery('group', groupRecord);
      approvalRecord.query();
      while (approvalRecord.next()) {
        approvalRecord.sysapproval = requestId;
        approvalRecord.setValue(
          'source_table',
          CMConstants.TABLES.DocumentRevision.name,
        );
        approvalRecord.setValue('document_id', documentId);
        approvalRecord.comments = comments;
        approvalRecord.update();
      }
    }

	var contractReq = new GlideRecord(CMConstants.TABLES.ContractRequest.name);
    if (contractReq.get(requestId)) {
        this.createPermissionsForAdhocApproval( 
            contractReq.getValue('folder_id'),
            'user_group',
            groupId,
            requestId,
            CMConstants.TABLES.ContractRequest.name,
        );
    }
  },
  isUserGroupHasRequiredRole: function(groupId, role) {
      var gr = new GlideRecord('sys_user_group');
      gr.addQuery('sys_id', groupId);
      gr.addQuery('roles', role);
      gr.query();
      if (gr.next()) {
        return true;
      } else {
		    return false;
      }
  },
  /**
   *
   * To check if approval can be initiated
   * @property {GlideRecord} requestRecord - Request Glide Record
   * */

  canInitiateApproval: function (requestRecord) {
    var isValidState = false;
    if (
      requestRecord.state ==
        CMConstants.CONTRACT_REQUEST_STATE.WORK_IN_PROGRESS ||
      requestRecord.state ==
        CMConstants.CONTRACT_REQUEST_STATE.AWAITING_APPROVAL
    )
      isValidState = true;
    var isValidContractStatus = false;
    if (
      requestRecord.contract_status !=
      CMConstants.CONTRACT_STATUS.PREPARING_FOR_SIGNATURE
    )
    isValidContractStatus = true;
	var userId = gs.getUserID();
	var isCollaborator = false;
	if (!gs.nil(requestRecord.getValue('additional_assignee_list'))) {
		isCollaborator = requestRecord.getValue('additional_assignee_list').indexOf(userId) != -1;
	}
	var isAssignedToOrManagerOrCollaborator =
            (userId === requestRecord.getValue('assigned_to') ||
            userId === requestRecord.getElement('assignment_group.manager').toString() ||
            isCollaborator);
    var currentUser = gs.getUser();
	var hasDoc = true;
	if (requestRecord.isValidField && requestRecord.isValidField('request_type') && requestRecord.getValue('request_type') == CMConstants.CONTRACT_REQUEST_TYPE.AMENDMENT_REQUEST) {
		hasDoc = new sn_cm_core.ContractDocumentUtils().hasDocumentByTarget(requestRecord.getUniqueValue());
	}
    return (
      isValidState &&
	  hasDoc &&
      isValidContractStatus &&
      currentUser.hasRole('sn_cm_core.contract_fulfiller') &&
	  isAssignedToOrManagerOrCollaborator &&
      requestRecord.canWrite()
    );
  },

  /**
   *
   * To check if the user is an approver.
   * @property {GlideRecord} - Document Revision Glide Record
   * */

  isApprover: function (revisionRecord, states) {
    var sysApproverGr = new GlideRecord(this.APPROVAL_TABLE);
    sysApproverGr.addQuery('document_id', revisionRecord.getUniqueValue());
    sysApproverGr
      .addQuery('approver', gs.getUserID());
    //   .addOrCondition(
    //     'sys_id',
    //     'IN',
    //     new global.ApprovalDelegationUtil().getAllMyApprovals(),
    //   );
    sysApproverGr.addQuery('state', 'IN', states);
    sysApproverGr.setLimit(1);
    sysApproverGr.query();
    return sysApproverGr.hasNext();
  },

  /**
   *
   * To check if the contract request has pending approvals.
   * @property {GlideRecord} requestRecord - Contract Request Glide Record
   * */
  hasPendingApprovals: function (requestRecord, approverId = '') {
    var approvalRecord = new GlideRecordSecure(this.APPROVAL_TABLE);
    approvalRecord.addQuery('sysapproval', requestRecord.getUniqueValue());
    approvalRecord.addQuery(
      'source_table',
      CMConstants.TABLES.DocumentRevision.name,
    );
    approvalRecord.addQuery('state', 'requested');
	if(approverId!=''){
        approvalRecord.addQuery('approver', approverId);
    }
    approvalRecord.setLimit(1);
    approvalRecord.query();
    return approvalRecord.hasNext();
  },

  /**
   *
   * To check if approval can be cancelled
   * @property {GlideRecord} requestRecord - Request Glide Record
   * */

  canCancelApproval: function (requestRecord) {
    var isValidState =
      requestRecord.state ==
      CMConstants.CONTRACT_REQUEST_STATE.AWAITING_APPROVAL;
	var userId = gs.getUserID();
	var isCollaborator = false;
	if (!gs.nil(requestRecord.getValue('additional_assignee_list'))) {
		isCollaborator = requestRecord.getValue('additional_assignee_list').indexOf(userId) != -1;
	}
	var isAssignedToOrManagerOrCollaborator =
            (userId === requestRecord.getValue('assigned_to') ||
            userId === requestRecord.getElement('assignment_group.manager').toString() ||
            isCollaborator);
    var currentUser = gs.getUser();
    return (
      isValidState &&
      currentUser.hasRole('sn_cm_core.contract_fulfiller') &&
	  isAssignedToOrManagerOrCollaborator &&
      requestRecord.canWrite() &&
      this.hasPendingApprovals(requestRecord)
    );
  },

  /**
   *
   * To get all the unique approvals
   * @property {SysID} recordId - Request record sysId
   * */
  getApprovals: function (recordId) {
    var sysIds = [];
    var approvalData = new GlideRecord(this.APPROVAL_TABLE);
    approvalData.addQuery('state', 'requested');
    approvalData.addQuery('sysapproval', recordId);
    approvalData.query();
    while (approvalData.next()) {
      var approvalObject = {};
      approvalObject.sys_id = approvalData.getUniqueValue();
      approvalObject.document_id = approvalData.getValue('document_id');
      sysIds.push(approvalObject);
    }
    var approvalObj = {};
    for (var i = 0; i < sysIds.length; i++) {
      approvalObj[sysIds[i]['document_id']] = sysIds[i];
      var filteredData = new Array();
      for (var key in approvalObj) filteredData.push(approvalObj[key]);
    }
    var uniqueApprovalArr = [];
    if (filteredData) {
      for (var k = 0; k < filteredData.length; k++) {
        uniqueApprovalArr.push(filteredData[k].sys_id.toString());
      }
    }
    var output = [];
    if (uniqueApprovalArr.length > 0) {
      var filteredApprovalData = new GlideRecordSecure(this.APPROVAL_TABLE);
      filteredApprovalData.addQuery(
        'sys_id',
        'IN',
        uniqueApprovalArr.toString(),
      );
      filteredApprovalData.query();

      while (filteredApprovalData.next()) {
        output.push({
          id: filteredApprovalData.getUniqueValue(),
          label: filteredApprovalData.getDisplayValue(),
        });
      }
    }
    return output;
  },

  /**
   *
   * To move the approval record to cancelled
   * @property {table} table - Table name of the request
   * @property {SysId} requestId - SysId of the request
   * @property {SysId} approvalId - SysId of the approval record
   * @property {string} comments - Comments of the approval record
   * */
  cancelApproval: function (table, requestId, approvalId, comments) {
    var requestRecord = new GlideRecord(
      CMConstants.TABLES.ContractRequest.name,
    );
    if (requestRecord.get(requestId)) {
      var approvalData = new GlideRecord(this.APPROVAL_TABLE);
      if (approvalData.get(approvalId)) {
        approvalData.state = 'cancelled';
        approvalData.comments = comments;
        approvalData.update();
      }
    }
  },

  /**
   *
   * To create the permissions record for external storage for given user or user group
   * @property {String} externalId - Id of the document uploaded on the external storage
   * @property {String} approvalBy - Approval By User or User Group
   * @property {sysId} userOrGroupId - SysId of the User or User Group
   * @property {sysId} sourceRecord - Source Record of the approval record
   * @property {Table} sourceTable - Source Table name
   * */
  createPermissionsForAdhocApproval: function (
    externalId,
    approvalBy,
    userOrGroupId,
    sourceRecord,
    sourceTable,
    documentGR,
  ) {
    var permissionMap = [];
    var contractRequest = new GlideRecord(sourceTable);
    contractRequest.get(sourceRecord);

    var storageDetails = '';
    var targetGr = new GlideRecord(contractRequest.table_name);
    if (targetGr.get(contractRequest.target_record)) {
        storageDetails = ContractExternalStorageUtils.getStorageDetails(targetGr, contractRequest);
    }
    if (approvalBy == 'user') {
      var email = ResolverUtils.getUserEmail(userOrGroupId);

      permissionMap.push({
        email: email[0],
        role: 'read',
        processType: 'scheduled',
        preserveHigherRole: true, 
      });
    } else if (approvalBy == 'user_group') {
      var emails = ResolverUtils.getGroupMemberEmails(userOrGroupId);

      emails.forEach(function (email) {
        permissionMap.push({
          email: email,
          role: 'read',
          processType: 'scheduled',
          preserveHigherRole: true,
        });
      });
    }
    permissionMap = ResolverUtils.convertPermissionArrayToMap(permissionMap);
    ExternalStoragePermissionProcessor.process(permissionMap, {
      itemId: externalId,
      sourceTable: contractRequest.getTableName(),
      sourceId: contractRequest.getUniqueValue(),
      provider: storageDetails.storageProvider,
    });
  },

  /**
   *
   * To update the state, contract status of a contract request based on approval status
   * @property {GlideRecord} approvalRecord - Approval record created for a revision.
   * */
  updateContractState: function (approvalRecord) {
    var requestRecord = new GlideRecord(
      CMConstants.TABLES.ContractRequest.name,
    );
    if (requestRecord.get(approvalRecord.sysapproval)) {
      var documentType = requestRecord.getValue('document_type');
      var approvalStates = [];

      if (documentType == CMConstants.DOCUMENT_TYPE.THIRD_PARTY_CONTRACT) {
        approvalStates = this.getAllApprovalStates(requestRecord);
      } else if (
        documentType == CMConstants.DOCUMENT_TYPE.TEMPLTE_BASED_CONTRACT
      ) {
        approvalStates.push(approvalRecord.getValue('state'));
      }
      var requestState = requestRecord.getValue('state');
      var contractStatus = requestRecord.getValue('contract_status');
      var expectedContractStatus = '';
      // Update the contract request state, contract status fields of the requestRecord.
      if (approvalStates.indexOf('requested') != -1) {
        if (contractStatus != CMConstants.CONTRACT_STATUS.AWAITING_APPROVAL) {
          requestRecord.state =
            CMConstants.CONTRACT_REQUEST_STATE.AWAITING_APPROVAL;
          requestRecord.contract_status =
            CMConstants.CONTRACT_STATUS.AWAITING_APPROVAL;
          requestRecord.update();
        }
        return;
      } else if (approvalStates.indexOf('rejected') != -1) {
        expectedContractStatus =
          CMConstants.CONTRACT_STATUS.DOCUMENT_APPROVAL_REJECTED;
      } else if (approvalStates.indexOf('approved') != -1) {
        expectedContractStatus = CMConstants.CONTRACT_STATUS.DOCUMENT_APPROVED;
      } else if (approvalStates.indexOf('cancelled') != -1) {
        expectedContractStatus =
          CMConstants.CONTRACT_STATUS.DOCUMENT_APPROVAL_CANCELLED;
      }
      if (expectedContractStatus != contractStatus) {
        requestRecord.contract_status = expectedContractStatus;
      }
      requestRecord.state = CMConstants.CONTRACT_REQUEST_STATE.WORK_IN_PROGRESS;
      requestRecord.update();
    }
  },

  /**
   *
   * To get the approval states of all the latest revisions of a contract request
   * @property {GlideRecord} requestRecord - Contract request record.
   * */
  getAllApprovalStates: function (requestRecord) {
    var approvalStates = [];
    // Get all the contract documents of the request
    var contractDocuments =
      new sn_cm_core.ContractDocumentUtils().getDocumentsByTarget(
        requestRecord.getUniqueValue(),
      );
    // Get approval status of all  the latest revisions.
    approvalStates = RecordList(contractDocuments).map(function (document) {
      if (!gs.nil(document.default_version)) {
        // Get the latest revision of each document and its approval status value
        var revisionGR = document.default_version.getRefRecord();
        if (revisionGR.getValue('ready'))
          return revisionGR.getValue('approval_status');
      }
    });
    return approvalStates;
  },
};
