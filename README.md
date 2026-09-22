(function process(
    /*RESTAPIRequest*/ request,
    /*RESTAPIResponse*/ response
) {

    var SERVICECALL_USER_TABLE =
        'x_1806573_servic_0_servicecall_user';

    var SERVICECALL_USER_ROLE =
        'x_1806573_servic_0.servicecall_user';

    var SERVICECALL_ADMIN_ROLE =
        'x_1806573_servic_0.servicecall_admin';


    /* -----------------------------------------
       AUTHENTICATED USER
    ----------------------------------------- */

    var userSysId =
        String(
            gs.getUserID() || ''
        );


    if (!userSysId) {

        response.setStatus(401);

        response.setBody({
            success: false,
            code: 'NO_AUTHENTICATED_USER',
            message:
                'Authenticated user could not be identified.'
        });

        return;
    }


    /* -----------------------------------------
       SYS_USER
    ----------------------------------------- */

    var userGR =
        new GlideRecord(
            'sys_user'
        );


    if (!userGR.get(userSysId)) {

        response.setStatus(404);

        response.setBody({
            success: false,
            code: 'USER_NOT_FOUND',
            message:
                'Authenticated ServiceNow user could not be found.'
        });

        return;
    }


    /* -----------------------------------------
       SERVICECALL USER ID
    ----------------------------------------- */

    var serviceCallId = '';


    var serviceCallUserGR =
        new GlideRecord(
            SERVICECALL_USER_TABLE
        );


    serviceCallUserGR.addQuery(
        'u_user',
        userSysId
    );

    serviceCallUserGR.setLimit(1);

    serviceCallUserGR.query();


    if (
        serviceCallUserGR.next()
    ) {

        serviceCallId =
            String(
                serviceCallUserGR.getValue(
                    'number'
                ) || ''
            );
    }


    /* -----------------------------------------
       STRICT ROLE CHECK
    ----------------------------------------- */

    function userHasAssignedRole(
        targetRoleName
    ) {

        var userRoleGR =
            new GlideRecord(
                'sys_user_has_role'
            );


        userRoleGR.addQuery(
            'user',
            userSysId
        );


        userRoleGR.addQuery(
            'role.name',
            targetRoleName
        );


        userRoleGR.setLimit(1);

        userRoleGR.query();


        return userRoleGR.hasNext();
    }


    /*
     * Check ServiceCall Admin FIRST.
     */
    var isServiceCallAdmin =
        userHasAssignedRole(
            SERVICECALL_ADMIN_ROLE
        );


    /*
     * A ServiceCall Admin is also a
     * ServiceCall User because our custom
     * admin role contains the custom user role.
     *
     * We make that relationship explicit here
     * rather than depending on OOB admin
     * behavior.
     */
    var isServiceCallUser =
        isServiceCallAdmin ||
        userHasAssignedRole(
            SERVICECALL_USER_ROLE
        );


    var allowed =
        isServiceCallUser ||
        isServiceCallAdmin;


    /* -----------------------------------------
       RESPONSE
    ----------------------------------------- */

    response.setStatus(200);

    response.setBody({

        success: true,

        code:
            'CURRENT_USER_RETRIEVED',

        user: {

            sys_id:
                userSysId,

            name:
                String(
                    userGR.getDisplayValue() ||
                    ''
                ),

            user_name:
                String(
                    userGR.getValue(
                        'user_name'
                    ) || ''
                ),

            email:
                String(
                    userGR.getValue(
                        'email'
                    ) || ''
                ),

            servicecall_id:
                serviceCallId
        },


        authorization: {

            allowed:
                allowed,

            is_servicecall_user:
                isServiceCallUser,

            is_servicecall_admin:
                isServiceCallAdmin
        }

    });

})(request, response);
