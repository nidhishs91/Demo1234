ipcMain.on(
    'servicecall-open-notification',
    (
        event,
        notificationData
    ) => {
 
        console.log(
            'ServiceCall notification OPEN request received:',
            notificationData
        );
 
 
        /*
         * Restore ServiceCall when the user
         * clicks a desktop notification.
         */
 
        showMainWindow();
 
 
        /*
         * Make sure the main ServiceCall
         * window is brought to the front.
         */
 
        if (
            mainWindow &&
            !mainWindow.isDestroyed()
        ) {
 
            if (
                mainWindow.isMinimized()
            ) {
 
                mainWindow.restore();
            }
 
 
            mainWindow.show();
 
            mainWindow.focus();
        }
    }
); 
