// /// Section 1: Toggler and Dynamic Content ///

// Attach openRespondPopup function to the window object for global scope
window.openRespondPopup = function(notificationId) {
    console.log(`Respond button clicked for notification ${notificationId}`); // Debug log to check if the button works

    // Show the pop-up and the overlay
    document.getElementById('respond-popup').style.display = 'block';
    document.getElementById('popup-overlay').style.display = 'block';

    // Add click event for Yes button
    document.getElementById('yes-btn').onclick = function() {
        console.log('Yes button clicked'); // Debug log for Yes button
        updateRespondStatus(notificationId, 'yes'); // User selected Yes
        closeRespondPopup(); // Close the pop-up after selection
    };

    // Add click event for No button
    document.getElementById('no-btn').onclick = function() {
        console.log('No button clicked'); // Debug log for No button
        updateRespondStatus(notificationId, 'no'); // User selected No
        closeRespondPopup(); // Close the pop-up after selection
    };
}

// Function to close the Respond pop-up
function closeRespondPopup() {
    console.log('Closing pop-up'); // Debug log for pop-up closing
    document.getElementById('respond-popup').style.display = 'none';
    document.getElementById('popup-overlay').style.display = 'none';
}

// Get the toggler tabs and notification container
const updatesTab = document.getElementById('updates-tab');
const eventsTab = document.getElementById('events-tab');
const notificationsContainer = document.getElementById('notifications-container-section');
const notificationsContent = document.getElementById('notifications-content');

// Set up event listener for each notification's "Respond" button
function setupRespondButtons() {
    document.querySelectorAll('.respond-btn').forEach(button => {
        button.addEventListener('click', function() {
            const notificationId = button.getAttribute('data-id');
            openRespondPopup(notificationId); // Call the global function
        });
    });
}

// Ensure that all event listeners and setups occur after the DOM is fully loaded
window.addEventListener('load', function() {
    setupRespondButtons();  // Attach event listeners to "Respond" buttons
});



// /// Section 2 ///

// Function to check response status and reflect the appropriate icon (check or X)
function checkResponseStatus(notificationId) {
    Fliplet.Session.get().then(function (session) {
        const userId = session.user.id; // Get logged-in user's ID
        Fliplet.DataSources.connect(899062).then(function (connection) {
            return connection.find({
                where: {
                    notification_id: notificationId,
                    user_id: userId
                }
            });
        }).then(function (records) {
            if (records.length > 0) {
                const response = records[0].data.response; // Get response value (yes or no)
                
                // Retrieve the notification icons container and insert the correct icon based on the response
                const notificationIconsContainer = document.querySelector(`#notification-${notificationId} .notification-icons`);
                if (notificationIconsContainer) {
                    if (response === 'yes') {
                        notificationIconsContainer.insertAdjacentHTML('beforeend', `<i class="fa fa-check icon-check" id="check-${notificationId}"></i>`);
                    } else if (response === 'no') {
                        notificationIconsContainer.insertAdjacentHTML('beforeend', `<i class="fa fa-times icon-x" id="x-${notificationId}"></i>`);
                    }
                } else {
                    console.error(`Error: Notification icons container not found for notification ID ${notificationId}`);
                }
            } else {
                console.log(`No response record found for notification ${notificationId} and user ${userId}.`);
            }
        }).catch(function (error) {
            console.error(`Error finding response status for notification ${notificationId}:`, error);
        });
    });
}




// /// Section 3 ///

// Function to switch between views and fetch notifications
function switchView(type) {
    notificationsContainer.style.opacity = 0;

    fetchNotifications(type).then((notifications) => {
        setTimeout(() => {
            if (notifications.length > 0) {
                notificationsContent.innerHTML = notifications.map(notification => {
                    // Initialize variables for icons and elements
                    let responseIcon = '';
                    let respondText = '';
                    let trashIcon = `<i class="fa fa-trash icon-trash" id="trash-${notification.id}" onclick="deleteNotification(${notification.id})"></i>`;
                    let urlLinkIcon = notification.data.url_link ? 
                        `<a href="${notification.data.url_link}" class="icon-link" target="_blank">
                            <i class="fa fa-external-link"></i>
                        </a>` : '';

                    // Check if a response was given by the user in 899062 data source
                    if (notification.data.response === 'yes') {
                        responseIcon = `<i class="fa fa-check icon-check" id="check-${notification.id}"></i>`;
                    } else if (notification.data.response === 'no') {
                        responseIcon = `<i class="fa fa-times icon-x" id="x-${notification.id}"></i>`;
                    } else {
                        // If no response yet, display "Reply " with a chat icon
                        if (notification.data.respond_request === 'request') {
                            respondText = `
                                <div class="reply-section">
                                    Reply: <i class="fa fa-comments icon-chat" id="chat-icon-${notification.id}" onclick="openRespondPopup(${notification.id})"></i>
                                </div>
                            `;
                        }
                    }

                    return `
                        <div class="notification-wrapper" id="notification-wrapper-${notification.id}">
                            ${trashIcon}
                            <div class="notification-item" id="notification-${notification.id}">
                                <div class="notification-content">
                                    <h3>${notification.data.notification_title}</h3>
                                    <p>${notification.data.notification_body}</p>
                                    ${respondText}
                                    <div class="notification-icons" style="display: flex; align-items: center; margin-top: 10px;">
                                        ${urlLinkIcon}
                                        ${responseIcon}
                                    </div>
                                </div>
                                <!-- Red dot for Unread notifications -->
                                <div class="notification-circle" id="circle-${notification.id}" style="background-color: red;"></div> 
                            </div>
                        </div>
                    `;
                }).join('');

                setupRespondButtons();  // Attach event listeners to "Respond" buttons

                // Check read/unread status and response for each notification
                notifications.forEach(notification => {
                    checkReadStatus(notification.id); // Check if it's read or not
                    checkResponseStatus(notification.id); // Check the response status and display check/X icon

                    // Add click event to toggle read/unread status
                    document.getElementById(`notification-${notification.id}`).addEventListener('click', function () {
                        toggleReadStatus(notification.id);
                    });
                });
            } else {
                notificationsContent.innerHTML = `No notifications for ${type}`;
            }
            notificationsContainer.style.opacity = 1;
        }, 300);
    });
}




// /// Section 4 ///

// Function to open the Respond pop-up
function openRespondPopup(notificationId) {
    console.log(`Respond button clicked for notification ${notificationId}`);

    // Show pop-up and overlay
    document.getElementById('respond-popup').style.display = 'block';
    document.getElementById('popup-overlay').style.display = 'block';

    // Yes button click event
    document.getElementById('yes-btn').onclick = function() {
        updateRespondStatus(notificationId, 'yes'); // User selected Yes
        closeRespondPopup(); // Close pop-up
    };

    // No button click event
    document.getElementById('no-btn').onclick = function() {
        updateRespondStatus(notificationId, 'no'); // User selected No
        closeRespondPopup(); // Close pop-up
    };
}

// Function to close the Respond pop-up
function closeRespondPopup() {
    document.getElementById('respond-popup').style.display = 'none';
    document.getElementById('popup-overlay').style.display = 'none';
}



// /// Section 5 ///

// Function to update response status in the data source and UI
function updateRespondStatus(notificationId, response) {
    console.log(`Updating response to ${response} for notification ${notificationId}`);

    // Retrieve button and icons for response
    const notificationIconsContainer = document.querySelector(`#notification-${notificationId} .notification-icons`);

    // Check if the notification icons container exists before inserting HTML
    if (!notificationIconsContainer) {
        console.error(`Error: Notification icons container not found for notification ID ${notificationId}.`);
        return; // Exit the function if the container is null
    }

    const checkIcon = document.getElementById(`check-${notificationId}`);
    const xIcon = document.getElementById(`x-${notificationId}`);

    // Connect to data source B to store the user's response
    Fliplet.DataSources.connect(899062).then(connection => {
        return connection.insert({
            notification_id: notificationId,
            response: response,
            first_read_at: new Date()
        }).then(() => {
            console.log(`Response updated for notification ${notificationId} to ${response}`);
            
            // Update UI based on the response
            if (response === 'yes') {
                // Add green check if not present
                if (!checkIcon) {
                    notificationIconsContainer.insertAdjacentHTML('beforeend', `<i class="fa fa-check icon-check" id="check-${notificationId}"></i>`);
                }
                // Remove red X if present
                if (xIcon) xIcon.remove();
            } else {
                // Add red X if not present
                if (!xIcon) {
                    notificationIconsContainer.insertAdjacentHTML('beforeend', `<i class="fa fa-times icon-x" id="x-${notificationId}"></i>`);
                }
                // Remove green check if present
                if (checkIcon) checkIcon.remove();
            }
        }).catch(error => {
            console.error('Error saving response:', error);
        });
    });
}



// /// Section 6 ///

// Function to check if a notification is read or not
function checkReadStatus(notificationId) {
    Fliplet.Session.get().then(function (session) {
        const userId = session.user.id; // Get logged-in user's ID
        Fliplet.DataSources.connect(899062).then(function (connection) {
            return connection.find({
                where: {
                    notification_id: notificationId,
                    user_id: userId
                }
            });
        }).then(function (records) {
            if (records.length > 0) {
                const currentView = records[0].data.current_view; // Get the current_view value
                console.log(`Current view for notification ${notificationId}: ${currentView}`);
                
                // Update the red dot based on current_view
                if (currentView === 'Unread') {
                    document.getElementById(`circle-${notificationId}`).style.display = 'block'; // Show red dot for Unread
                } else if (currentView === 'Read') {
                    document.getElementById(`circle-${notificationId}`).style.display = 'none'; // Hide red dot for Read
                }
            } else {
                console.log(`No read status found for notification ${notificationId} and user ${userId}. Defaulting to unread.`);
                document.getElementById(`circle-${notificationId}`).style.display = 'block'; // Default to show red dot for Unread
            }
        }).catch(function (error) {
            console.error(`Error finding read status for notification ${notificationId}:`, error);
        });
    });
}

// /// Section 7 ///

// Function to toggle the read status on touch
function toggleReadStatus(notificationId) {
    console.log(`Toggling read status for notification ID: ${notificationId}`);

    Fliplet.Session.get().then(function (session) {
        if (session && session.user && session.user.id) {
            const userId = session.user.id; // Get logged-in user's ID
            const deviceInfo = navigator.userAgent || 'Unknown Device'; // Device info

            // Instantly update the UI for better responsiveness
            const notificationCircle = document.getElementById(`circle-${notificationId}`);
            const isUnread = notificationCircle.style.display !== 'none'; // Check if it's unread

            if (isUnread) {
                notificationCircle.style.display = 'none'; // Mark as read instantly in UI
                console.log(`Notification ${notificationId} marked as Read instantly.`);
            } else {
                notificationCircle.style.display = 'block'; // Mark as unread instantly in UI
                console.log(`Notification ${notificationId} marked as Unread instantly.`);
            }

            // Now update the data source in the background
            Fliplet.DataSources.connect(899062).then(function (connection) {
                return connection.find({
                    where: {
                        notification_id: notificationId,
                        user_id: userId
                    }
                }).then(function (records) {
                    if (records.length > 0) {
                        const currentView = records[0].data.current_view;

                        if (currentView === 'Read') {
                            connection.update(records[0].id, {
                                current_view: 'Unread',
                                action_taken: 'Unread',
                                device_info: deviceInfo
                            });
                        } else {
                            connection.update(records[0].id, {
                                current_view: 'Read',
                                action_taken: 'Read',
                                device_info: deviceInfo
                            });
                        }
                    } else {
                        connection.insert({
                            notification_id: notificationId,
                            user_id: userId,
                            first_read_at: new Date(),
                            current_view: 'Unread',
                            device_info: deviceInfo
                        });
                    }
                });
            }).catch(function (error) {
                console.error(`Error updating read status for notification ${notificationId}:`, error);
            });
        } else {
            console.error(`Error: User not logged in or session data is missing.`);
        }
    });
}



// /// Section 8 ///

// Function to fetch and sort notifications from data source A based on type
function fetchNotifications(type) {
    return Fliplet.DataSources.connect(899061).then(function (connection) {
        return connection.find({
            where: { notification_type: type } // Filter by 'Update' or 'Event'
        }).then(function (notifications) {
            // Sort by date and time, newest first
            return notifications.sort((a, b) => {
                const dateA = new Date(`${a.data.date} ${a.data.time}`);
                const dateB = new Date(`${b.data.date} ${b.data.time}`);
                return dateB - dateA; // Newest first
            });
        });
    });
}

// /// Section 9 ///

// Set default view to 'Updates' (Fetch and display "Update" notifications)
window.onload = function() {
    updatesTab.classList.add('selected');
    switchView('Update');
    countUnreadNotifications('Update'); // Check for unread "Updates"
    countUnreadNotifications('Event');  // Check for unread "Events"
};

// /// Section 9 ///

// Event listener for Updates tab
updatesTab.addEventListener('click', function() {
    // Remove 'selected' class from the other tab
    eventsTab.classList.remove('selected');
    // Add 'selected' class to this tab
    updatesTab.classList.add('selected');

    // Fetch and display "Update" notifications on click
    switchView('Update');

    // Poll the unread count for Updates only when the user clicks
    countUnreadNotifications('Update'); // Trigger polling when clicked
});

// Event listener for Events tab
eventsTab.addEventListener('click', function() {
    // Remove 'selected' class from the other tab
    updatesTab.classList.remove('selected');
    // Add 'selected' class to this tab
    eventsTab.classList.add('selected');

    // Fetch and display "Event" notifications on click
    switchView('Event');

    // Poll the unread count for Events only when the user clicks
    countUnreadNotifications('Event'); // Trigger polling when clicked
});


// /// Section 10 ///

// Function to check if a notification is read or not
function checkReadStatus(notificationId) {
    Fliplet.Session.get().then(function (session) {
        const userId = session.user.id; // Get logged-in user's ID
        Fliplet.DataSources.connect(899062).then(function (connection) {
            return connection.find({
                where: {
                    notification_id: notificationId,
                    user_id: userId
                }
            });
        }).then(function (records) {
            if (records.length > 0) {
                const currentView = records[0].data.current_view; // Get the current_view value
                console.log(`Current view for notification ${notificationId}: ${currentView}`);
                
                // Update the red dot based on current_view
                if (currentView === 'Unread') {
                    document.getElementById(`circle-${notificationId}`).style.display = 'block'; // Show red dot for Unread
                } else if (currentView === 'Read') {
                    document.getElementById(`circle-${notificationId}`).style.display = 'none'; // Hide red dot for Read
                }
            } else {
                console.log(`No read status found for notification ${notificationId} and user ${userId}. Defaulting to unread.`);
                document.getElementById(`circle-${notificationId}`).style.display = 'block'; // Default to show red dot for Unread
            }
        }).catch(function (error) {
            console.error(`Error finding read status for notification ${notificationId}:`, error);
        });
    });
}




// /// Section 11 ///


// Function to toggle the read status on touch and trigger polling
function toggleReadStatus(notificationId) {
    console.log(`Toggling read status for notification ID: ${notificationId}`);

    // Select the notification element
    const notificationElement = document.getElementById(`notification-${notificationId}`);
    
    notificationElement.addEventListener('click', function (event) {
        // Calculate the position of the click within the notification
        const clickPosition = event.clientX - notificationElement.getBoundingClientRect().left;
        const notificationWidth = notificationElement.offsetWidth;

        // Only proceed if the click is within the right 30% of the notification width
        if (clickPosition >= notificationWidth * 0.7) {
            console.log(`Click within the right 30% detected. Proceeding with toggle for notification ${notificationId}.`);
            
            Fliplet.Session.get().then(function (session) {
                if (session && session.user && session.user.id) {
                    const userId = session.user.id; // Get logged-in user's ID
                    const deviceInfo = navigator.userAgent || 'Unknown Device'; // Device info
                    const userName = session.user.name || 'Unknown'; // Get logged-in user's name (if available)

                    // Instantly update the UI to show read/unread status for better responsiveness
                    const notificationCircle = document.getElementById(`circle-${notificationId}`);
                    const isUnread = notificationCircle.style.display !== 'none'; // Check if it's unread

                    // Toggle display for the red dot in UI
                    if (isUnread) {
                        notificationCircle.style.display = 'none'; // Mark as read instantly in UI
                        console.log(`Notification ${notificationId} marked as Read instantly in the UI.`);
                    } else {
                        notificationCircle.style.display = 'block'; // Mark as unread instantly in UI
                        console.log(`Notification ${notificationId} marked as Unread instantly in the UI.`);
                    }

                    // Update the data source in the background
                    Fliplet.DataSources.connect(899062).then(function (connection) {
                        return connection.find({
                            where: {
                                notification_id: notificationId,
                                user_id: userId
                            }
                        }).then(function (records) {
                            if (records.length > 0) {
                                const currentView = records[0].data.current_view;
                                if (currentView === 'Read') {
                                    connection.update(records[0].id, {
                                        current_view: 'Unread',
                                        action_taken: 'Unread',
                                        device_info: deviceInfo,
                                        first_read_at: records[0].data.first_read_at || new Date()
                                    }).then(function () {
                                        console.log(`Updated current_view to Unread for notification ${notificationId}.`);
                                    });
                                } else {
                                    connection.update(records[0].id, {
                                        current_view: 'Read',
                                        action_taken: 'Read',
                                        device_info: deviceInfo,
                                        first_read_at: records[0].data.first_read_at || new Date()
                                    }).then(function () {
                                        console.log(`Updated current_view to Read for notification ${notificationId}.`);
                                    });
                                }
                            } else {
                                connection.insert({
                                    notification_id: notificationId,
                                    user_id: userId,
                                    first_read_at: new Date(),
                                    current_view: 'Unread',
                                    device_info: deviceInfo,
                                    user_name: userName,
                                    action_taken: 'Unread'
                                }).then(function () {
                                    console.log(`Inserted record for notification ${notificationId} with current_view as Unread.`);
                                });
                            }
                        });
                    });
                } else {
                    console.error(`Error: User not logged in or session data is missing.`);
                }
            });
        } else {
            console.log(`Click outside the right 30% ignored for notification ${notificationId}.`);
        }
    });
}




// /// Section 12 ///

// Function to count unread notifications for a given type (Update/Event)
function countUnreadNotifications(type) {
    Fliplet.Session.get().then(function (session) {
        const userId = session.user.id; // Get logged-in user's ID
        console.log(`Counting unread notifications for user ID: ${userId} and type: ${type}`);
        
        // Connect to A (NotificationCenter) and fetch notifications of the given type
        Fliplet.DataSources.connect(899061).then(function (connectionA) {
            return connectionA.find({
                where: {
                    notification_type: type // Filter notifications by type: "Update" or "Event"
                }
            }).then(function (notifications) {
                console.log(`Total ${type} notifications found: ${notifications.length}`);
                
                // If no notifications of the type, update the dot to show 0
                if (notifications.length === 0) {
                    console.log(`No ${type} notifications found.`);
                    updateNotificationDot(type, 0); // No notifications, no unread count
                    return;
                }

                let unreadCount = 0; // Initialize the unread count
                
                // Prepare an array of promises to handle checking the unread status for each notification
                const promises = notifications.map(function(notification) {
                    // Check unread status from data source B (NotificationReadTracking)
                    return Fliplet.DataSources.connect(899062).then(function (connectionB) {
                        return connectionB.find({
                            where: {
                                notification_id: notification.id,
                                user_id: userId,
                                current_view: 'Unread' // Check for unread status
                            }
                        }).then(function (readRecords) {
                            if (readRecords.length > 0) {
                                // If any records are unread, increment the unread count
                                unreadCount++;
                                console.log(`Notification ID ${notification.id} is unread. Incrementing count.`);
                            }
                        });
                    });
                });
                
                // Use Promise.all to ensure all promises resolve before updating the dot
                Promise.all(promises).then(function() {
                    console.log(`Final unread count for ${type}: ${unreadCount}`);
                    updateNotificationDot(type, unreadCount); // Update the dot with the final count
                }).catch(function(error) {
                    console.error(`Error processing unread notifications for ${type}:`, error);
                });
            });
        });
    });
}




// /// Section 13 ///

// Function to update the notification dot with the unread count for each tab (Updates/Events)
function updateNotificationDot(type, count) {
    let dotElement;

    // Select the correct dot element based on the type (Updates or Events)
    if (type === 'Update') {
        dotElement = document.getElementById('updates-notification-dot');
    } else if (type === 'Event') {
        dotElement = document.getElementById('events-notification-dot');
    }

    // Show the dot only if there are unread notifications
    if (count > 0) {
        dotElement.innerText = count; // Display the count inside the dot
        dotElement.style.display = 'flex'; // Show the dot only if there are unread notifications
        console.log(`Showing ${type} dot with count: ${count}`);
    } else {
        dotElement.style.display = 'none'; // Hide the dot if there are no unread notifications
        console.log(`No unread ${type} notifications. Hiding the dot.`);
    }
}

// /// Section 14 ///

// Function to start polling for live updates every X seconds (adjust the interval as needed)
function startPolling(interval = 50000) { // Polling every 5 seconds
    setInterval(function() {
        console.log("Polling for live unread count updates...");

        // Count unread notifications for "Updates" and "Events"
        countUnreadNotifications('Update'); // Updates red dot for "Updates"
        countUnreadNotifications('Event');  // Updates red dot for "Events"

        // No need to refresh the entire notification list or UI
    }, interval);
}

// Start the polling when the app is loaded or the user switches tabs
startPolling(); // Start polling for live unread count updates



// /// Section 15 ///

// Function to open the Respond pop-up
function openRespondPopup(notificationId) {
    console.log(`Respond button clicked for notification ${notificationId}`); // Debug log to check if the button works

    // Show the pop-up and the overlay
    document.getElementById('respond-popup').style.display = 'block';
    document.getElementById('popup-overlay').style.display = 'block';

    // Add click event for Yes button
    document.getElementById('yes-btn').onclick = function() {
        console.log('Yes button clicked'); // Debug log for Yes button
        updateRespondStatus(notificationId, 'yes'); // User selected Yes
        closeRespondPopup(); // Close the pop-up after selection
    };

    // Add click event for No button
    document.getElementById('no-btn').onclick = function() {
        console.log('No button clicked'); // Debug log for No button
        updateRespondStatus(notificationId, 'no'); // User selected No
        closeRespondPopup(); // Close the pop-up after selection
    };
}

// Function to close the Respond pop-up
function closeRespondPopup() {
    console.log('Closing pop-up'); // Debug log for pop-up closing
    document.getElementById('respond-popup').style.display = 'none';
    document.getElementById('popup-overlay').style.display = 'none';
}

// /// Section 16 ///


// Function to update Respond status and save to data source
function updateRespondStatus(notificationId, response) {
    console.log(`Updating response to ${response} for notification ${notificationId}`); // Debug log for updating response

    Fliplet.Session.get().then(function (session) {
        const userId = session.user.id; // Get logged-in user's ID
        
        Fliplet.DataSources.connect(899062).then(function (connection) {
            // Check if a record exists for this notification and user
            connection.find({
                where: {
                    notification_id: notificationId,
                    user_id: userId
                }
            }).then(function (records) {
                if (records.length > 0) {
                    // Record exists, so we update it with the new response
                    const recordId = records[0].id;
                    connection.update(recordId, {
                        response: response // Update the response to yes or no
                    }).then(function () {
                        console.log(`Response for notification ${notificationId} updated to ${response}`);
                        
                        // Update the UI based on the response
                        const respondButton = document.getElementById(`respond-btn-${notificationId}`);
                        const checkIcon = document.getElementById(`check-${notificationId}`);
                        const xIcon = document.getElementById(`x-${notificationId}`);

                        if (response === 'yes') {
                            // Show green check
                            if (!checkIcon) {
                                respondButton.insertAdjacentHTML('afterend', `<i class="fa fa-check" id="check-${notificationId}" style="color: green; margin-left: 10px;"></i>`);
                            }
                            // Remove the red X if it was there
                            if (xIcon) xIcon.remove();
                        } else {
                            // Show red X
                            if (!xIcon) {
                                respondButton.insertAdjacentHTML('afterend', `<i class="fa fa-times" id="x-${notificationId}" style="color: red; margin-left: 10px;"></i>`);
                            }
                            // Remove the green check if it was there
                            if (checkIcon) checkIcon.remove();
                        }
                    }).catch(function (error) {
                        console.error('Error updating response:', error);
                    });
                } else {
                    // No record exists, so we insert a new one
                    connection.insert({
                        notification_id: notificationId,
                        user_id: userId,
                        response: response,
                        first_read_at: new Date() // Optional: Save the time of interaction
                    }).then(function () {
                        console.log(`New record inserted with response ${response} for notification ${notificationId}`);
                    }).catch(function (error) {
                        console.error('Error inserting response:', error);
                    });
                }
            }).catch(function (error) {
                console.error('Error finding existing record:', error);
            });
        });
    });
}
