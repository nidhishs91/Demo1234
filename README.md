(function process(
    /*RESTAPIRequest*/
    request,
    /*RESTAPIResponse*/
    response
) {

    var userId =
        gs.getUserID();


    if (!userId) {

        response.setStatus(401);

        response.setBody({
            success: false,
            code: 'NO_AUTHENTICATED_USER',
            message: 'Authenticated user could not be identified.'
        });

        return;
    }


    var callSysId =
        (request.queryParams.call_sys_id || '')
        .toString()
        .trim();


    if (!callSysId) {

        response.setStatus(400);

        response.setBody({
            success: false,
            code: 'CALL_ID_REQUIRED',
            message: 'Call ID is required.'
        });

        return;
    }


    /* ---------------------------------------------------
       CURRENT PARTICIPANT
    --------------------------------------------------- */

    var participantGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_participant'
        );


    participantGR.addQuery(
        'u_call',
        callSysId
    );

    participantGR.addQuery(
        'u_user',
        userId
    );

    participantGR.setLimit(1);

    participantGR.query();


    if (!participantGR.next()) {

        response.setStatus(403);

        response.setBody({
            success: false,
            code: 'NOT_CALL_PARTICIPANT',
            message: 'You are not a participant in this call.'
        });

        return;
    }


    /* ---------------------------------------------------
       CALL
    --------------------------------------------------- */

    var callGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_call'
        );


    if (!callGR.get(callSysId)) {

        response.setStatus(404);

        response.setBody({
            success: false,
            code: 'CALL_NOT_FOUND',
            message: 'Call was not found.'
        });

        return;
    }


    /* ---------------------------------------------------
       OWNER / ROLE
    --------------------------------------------------- */

    var originalCallerId =
        callGR.getValue(
            'u_original_caller'
        ) || '';


    var ownerValue =
        participantGR.getValue(
            'u_is_owner'
        );


    var participantIsOwner =
        ownerValue === '1' ||
        ownerValue === 'true';


    /*
     * Keep original caller as a fallback.
     * This protects older call records if
     * u_is_owner was not populated.
     */
    var isOwner =
        participantIsOwner ||
        originalCallerId === userId;


    var participantRole =
        participantGR.getValue(
            'u_role'
        ) || 'participant';


    /* ---------------------------------------------------
       OTHER USER
       Preserve existing 1:1 response fields
    --------------------------------------------------- */

    var otherUserId =
        '';

    var otherUserName =
        '';

    var otherUserDepartment =
        '';


    var otherParticipantGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_participant'
        );


    otherParticipantGR.addQuery(
        'u_call',
        callSysId
    );

    otherParticipantGR.addQuery(
        'u_user',
        '!=',
        userId
    );

    otherParticipantGR.orderBy(
        'sys_created_on'
    );

    otherParticipantGR.setLimit(1);

    otherParticipantGR.query();


    if (otherParticipantGR.next()) {

        otherUserId =
            otherParticipantGR.getValue(
                'u_user'
            ) || '';


        var otherUserGR =
            new GlideRecord(
                'sys_user'
            );


        if (
            otherUserId &&
            otherUserGR.get(
                otherUserId
            )
        ) {

            otherUserName =
                otherUserGR.getDisplayValue(
                    'name'
                ) || '';


            otherUserDepartment =
                otherUserGR.getDisplayValue(
                    'department'
                ) || '';
        }
    }


    /* ---------------------------------------------------
       ACTIVE PARTICIPANTS
    --------------------------------------------------- */

    var participants = [];


    var activeParticipantGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_participant'
        );


    activeParticipantGR.addQuery(
        'u_call',
        callSysId
    );

    activeParticipantGR.addQuery(
        'u_status',
        'IN',
        'connected,ringing,invited'
    );

    activeParticipantGR.orderBy(
        'sys_created_on'
    );

    activeParticipantGR.query();


    while (
        activeParticipantGR.next()
    ) {

        var activeUserId =
            activeParticipantGR.getValue(
                'u_user'
            ) || '';


        var activeUserName =
            'Unknown User';

        var activeUserDepartment =
            '';


        var activeUserGR =
            new GlideRecord(
                'sys_user'
            );


        if (
            activeUserId &&
            activeUserGR.get(
                activeUserId
            )
        ) {

            activeUserName =
                activeUserGR.getDisplayValue(
                    'name'
                ) ||
                'Unknown User';


            activeUserDepartment =
                activeUserGR.getDisplayValue(
                    'department'
                ) || '';
        }


        var activeOwnerValue =
            activeParticipantGR.getValue(
                'u_is_owner'
            );


        var activeIsOwner =
            activeOwnerValue === '1' ||
            activeOwnerValue === 'true' ||
            activeUserId === originalCallerId;


        participants.push({

            user_sys_id: activeUserId,

            name: activeUserName,

            department: activeUserDepartment,

            status: activeParticipantGR.getValue(
                'u_status'
            ) || '',

            role: activeParticipantGR.getValue(
                'u_role'
            ) || 'participant',

            is_owner: activeIsOwner

        });
    }

    /* ---------------------------------------------------
   ACTIVE RECORDING
--------------------------------------------------- */

    var recordingActive =
        false;

    var activeRecordingSysId =
        '';

    var recordingStartedAt =
        '';

    var recordingStartedBy =
        '';


    /*
     * A recording is considered actively capturing
     * only while its status is "recording".
     *
     * "processing" means capture has already stopped
     * and the file is being finalized.
     */
    var recordingGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_recording'
        );


    recordingGR.addQuery(
        'u_call',
        callSysId
    );

    recordingGR.addQuery(
        'u_status',
        'recording'
    );

    recordingGR.orderByDesc(
        'u_started_at'
    );

    recordingGR.setLimit(1);

    recordingGR.query();


    if (recordingGR.next()) {

        recordingActive =
            true;


        activeRecordingSysId =
            recordingGR.getUniqueValue();


        recordingStartedAt =
            recordingGR.getValue(
                'u_started_at'
            ) || '';


        recordingStartedBy =
            recordingGR.getValue(
                'u_recorded_by'
            ) || '';
    }


    /* ---------------------------------------------------
       RESPONSE
    --------------------------------------------------- */

    response.setStatus(200);


    response.setBody({

        success: true,

        code: 'CALL_STATUS_OK',


        call_sys_id: callSysId,


        call_number: callGR.getDisplayValue(
            'number'
        ) || '',


        conference_id: callGR.getValue(
            'u_conference_id'
        ) || '',


        state: callGR.getValue(
            'u_state'
        ) || '',

        started_at: callGR.getValue(
            'u_started_at'
        ) || '',

        answered_at: callGR.getValue(
            'u_answered_at'
        ) || '',


        ended_at: callGR.getValue(
            'u_ended_at'
        ) || '',


        participant_status: participantGR.getValue(
            'u_status'
        ) || '',


        participant_role: participantRole,


        is_owner: isOwner,


        participant_count: participants.length,


        participants: participants,


        /*
         * Current recording state.
         *
         * This allows every participant,
         * including somebody who joined late,
         * to know whether this call is
         * currently being recorded.
         */
        recording_active: recordingActive,


        active_recording_sys_id: activeRecordingSysId,


        recording_started_at: recordingStartedAt,


        recording_started_by: recordingStartedBy,


        /*
         * Preserve these existing fields
         * for current desktop code.
         */
        other_user_sys_id: otherUserId,


        other_user_name: otherUserName,


        other_user_department: otherUserDepartment

    });


})(
    request,
    response
);
