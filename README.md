async function createWindow() {

    mainWindow = new BrowserWindow({
        width: 900,
        height: 650,
        minWidth: 700,
        minHeight: 500,
        title: 'ServiceCall Desktop',

        webPreferences: {
            preload: path.join(
                __dirname,
                'preload.js'
            ),

            contextIsolation: true,
            nodeIntegration: false
        }
    });


    /*
     * -----------------------------------------
     * RESOLVE STARTUP AUTHENTICATION
     * -----------------------------------------
     */

    const startupState =
        await resolveStartupAuthentication();


    console.log(
        'ServiceCall startup state:',
        startupState
    );


    /*
     * -----------------------------------------
     * AUTHORIZED USER
     * -----------------------------------------
     */

    if (
        startupState.state === 'ready'
    ) {

        await mainWindow.loadFile(
            'index.html'
        );


        /*
         * Start background ServiceCall services
         * ONLY after authorization succeeds.
         */

        await startHeartbeatLoop();

        startIncomingCallLoop();

        startOutgoingCallLoop();

        startAuthorizationMonitor();
    }


    /*
     * -----------------------------------------
     * LOGIN / ACCESS GATE
     * -----------------------------------------
     */

    else {

        await mainWindow.loadFile(
            path.join(
                'auth',
                'auth-gate.html'
            )
        );
    }


    /*
     * -----------------------------------------
     * WINDOW CLOSE
     * -----------------------------------------
     */

    mainWindow.on(
        'close',
        (event) => {

            if (!isQuitting) {

                event.preventDefault();

                mainWindow.hide();
            }
        }
    );
}
