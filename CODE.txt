#include <iostream>
#include <fstream>
#include <ctime>
#include <vector>
#include <string>
#include <limits>
#include <queue>         // For std::queue
#include <unordered_map>   // For std::unordered_map
#include <thread>          // For std::thread
#include <mutex>           // For std::mutex, std::lock_guard
#include <chrono>          // For std::chrono
#include <atomic>          // For std::atomic<bool>
#include <sstream>         // For std::stringstream
#include <set>             // For std::set (today's unique attendees)
#include <list>            // For std::list (recent attendance activity)
#include <algorithm>       // For std::sort
#include <filesystem>      // For std::filesystem::create_directory (C++17)
#include <iomanip>         // For std::put_time

#include <opencv2/opencv.hpp>
#include <opencv2/highgui.hpp>
#include <opencv2/imgproc.hpp>

#ifdef _WIN32
#include <conio.h>
#else
#include <termios.h>
#include <unistd.h>
#endif

// Use std::filesystem for directory creation (C++17 standard)
namespace fs = std::filesystem;


using namespace std;
using namespace cv;

// --- Global Data Structures and Synchronization Primitives ---
// Queue for asynchronous attendance logging (for combined log file)
queue<string> g_attendanceLogQueue;
mutex g_logQueueMutex; // Mutex to protect access to the queue
condition_variable g_logQueueCv; // Condition variable to signal the logging thread

// Atomic flag to control the logging thread's lifecycle
atomic<bool> g_loggingThreadRunning(true);

// Unordered map to store last marked attendance time for cooldown
// Key: Combined name and ID (e.g., "JohnDoe_12345"), Value: timestamp of last attendance
unordered_map<string, time_t> g_attendanceCooldownCache;
mutex g_cooldownCacheMutex; // Mutex to protect access to the cooldown cache

// Data structures for overall attendance statistics (persists during program run)
// Key: Combined name and ID (e.g., "JohnDoe_12345"), Value: Total attendance count
unordered_map<string, int> g_studentTotalAttendance;
unordered_map<string, int> g_teacherTotalAttendance;
mutex g_totalAttendanceMutex; // Mutex for total attendance maps

// List for recent attendance activity (fixed size)
list<string> g_recentAttendance;
mutex g_recentAttendanceMutex; // Mutex for recent attendance list
const int MAX_RECENT_ATTENDEES = 5; // Max number of recent attendees to show

// Cooldown period (in seconds)
const int ATTENDANCE_COOLDOWN_SECONDS = 5; // Set to 5 seconds for easier testing

// --- Utility Functions ---
void clearScreen() {
#ifdef _WIN32
    system("cls");
#else
    system("clear");
#endif
}

// Function to create a directory if it doesn't exist (using std::filesystem)
void createDirectoryIfNotExists(const string& path) {
    if (!fs::exists(path)) {
        try {
            fs::create_directory(path);
            cout << "Created directory: " << path << endl;
        }
        catch (const fs::filesystem_error& e) {
            cerr << "Error creating directory " << path << ": " << e.what() << endl;
        }
    }
}

void returnToMenu() {
    cout << "\nPress any key to return to the menu...";
#ifdef _WIN32
    _getch();
#else
    // Custom implementation for non-Windows to mimic _getch()
    // Sets terminal to raw mode to read single char without enter
    struct termios oldt, newt;
    tcgetattr(STDIN_FILENO, &oldt); // Save current terminal settings
    newt = oldt;
    newt.c_lflag &= ~(ICANON | ECHO); // Disable canonical mode (line buffering) and echoing
    tcsetattr(STDIN_FILENO, TCSANOW, &newt); // Apply new settings
    getchar(); // Read a single character
    tcsetattr(STDIN_FILENO, TCSANOW, &oldt); // Restore old terminal settings
#endif
}

// Function to get current time as a formatted string (YYYY-MM-DD HH:MM:SS)
string getCurrentFormattedTime() {
    auto now = chrono::system_clock::now();
    time_t tt = chrono::system_clock::to_time_t(now);
    tm localTime;
#ifdef _WIN32
    localtime_s(&localTime, &tt);
#else
    localtime_r(&tt, &localTime); // POSIX version
#endif
    char buf[80];
    strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", &localTime);
    return string(buf);
}

// Function to get current date as a formatted string (YYYY-MM-DD)
string getCurrentFormattedDate() {
    auto now = chrono::system_clock::now();
    time_t tt = chrono::system_clock::to_time_t(now);
    tm localTime;
#ifdef _WIN32
    localtime_s(&localTime, &tt);
#else
    localtime_r(&tt, &localTime);
#endif
    char buf[80];
    strftime(buf, sizeof(buf), "%Y-%m-%d", &localTime);
    return string(buf);
}

// --- Asynchronous Logging Thread Function ---
void logWriterFunction(const string& combinedLogFileName) {
    ofstream combinedLogFile(combinedLogFileName, ios::app);
    if (!combinedLogFile.is_open()) {
        cerr << "Error: Could not open combined attendance log file for writing: " << combinedLogFileName << endl;
        g_loggingThreadRunning = false; // Stop thread if file can't be opened
        return;
    }

    while (g_loggingThreadRunning) {
        string logEntry;
        {
            unique_lock<mutex> lock(g_logQueueMutex);
            // Wait until there's data in the queue or the thread is signaled to stop
            g_logQueueCv.wait(lock, [&] { return !g_attendanceLogQueue.empty() || !g_loggingThreadRunning; });

            if (!g_attendanceLogQueue.empty()) {
                logEntry = g_attendanceLogQueue.front();
                g_attendanceLogQueue.pop();
            }
            else if (!g_loggingThreadRunning) {
                break; // Exit if thread is stopping and queue is empty
            }
        }

        if (!logEntry.empty()) {
            combinedLogFile << logEntry << endl;
            combinedLogFile.flush(); // Ensure data is written to disk immediately
        }
    }

    // Process any remaining items in the queue before exiting
    // This loop ensures no log entries are lost if the program exits gracefully
    while (true) {
        string logEntry;
        {
            lock_guard<mutex> lock(g_logQueueMutex);
            if (g_attendanceLogQueue.empty()) {
                break;
            }
            logEntry = g_attendanceLogQueue.front();
            g_attendanceLogQueue.pop();
        }
        combinedLogFile << logEntry << endl;
    }
    combinedLogFile.close();
}

// --- Binary Search Tree for Attendance Summary (by date) ---
struct AttendanceSummaryNode {
    string date;
    int count;
    AttendanceSummaryNode* left;
    AttendanceSummaryNode* right;

    AttendanceSummaryNode(string d, int c = 1) : date(d), count(c), left(nullptr), right(nullptr) {}
};

class AttendanceSummaryBST {
private:
    AttendanceSummaryNode* root;

    // Helper for recursive insertion
    AttendanceSummaryNode* insertRecursive(AttendanceSummaryNode* node, string date, int count) {
        if (node == nullptr) {
            return new AttendanceSummaryNode(date, count);
        }

        if (date < node->date) {
            node->left = insertRecursive(node->left, date, count);
        }
        else if (date > node->date) {
            node->right = insertRecursive(node->right, date, count);
        }
        else {
            // Date already exists, update count
            node->count += count;
        }
        return node;
    }

    // Helper for recursive in-order traversal (to print sorted)
    void inOrderTraversalRecursive(AttendanceSummaryNode* node) const {
        if (node == nullptr) {
            return;
        }
        inOrderTraversalRecursive(node->left);
        cout << "\t" << node->date << ": " << node->count << " entries\n";
        inOrderTraversalRecursive(node->right);
    }

    // Helper for recursive deletion (to free memory)
    void deleteTreeRecursive(AttendanceSummaryNode* node) {
        if (node == nullptr) {
            return;
        }
        deleteTreeRecursive(node->left);
        deleteTreeRecursive(node->right);
        delete node;
    }

public:
    AttendanceSummaryBST() : root(nullptr) {}

    // Destructor to free memory
    ~AttendanceSummaryBST() {
        deleteTreeRecursive(root);
    }

    void insert(string date, int count = 1) {
        root = insertRecursive(root, date, count);
    }

    void printSortedSummary() const {
        inOrderTraversalRecursive(root);
    }

    bool isEmpty() const {
        return root == nullptr;
    }
};


// --- Attendance Marking and Display Functions ---

// Helper function to mark attendance (now uses queue for async logging and cooldown cache)
// IMPORTANT: Now writes to both the specific log file AND the combined log via queue.
void markAttendanceInternal(const string& name, const string& id, const string& specificLogFileName,
    unordered_map<string, int>& totalAttendanceMap) {
    string uniqueId = name + "_" + id; // Combine name and ID for unique identifier

    // --- Cooldown check using unordered_map ---
    {
        lock_guard<mutex> lock(g_cooldownCacheMutex);
        auto it = g_attendanceCooldownCache.find(uniqueId);
        time_t now = time(0);

        if (it != g_attendanceCooldownCache.end()) {
            double timeDiff = difftime(now, it->second);
            if (timeDiff < ATTENDANCE_COOLDOWN_SECONDS) {
                cout << "Attendance for " << name << " (" << id << ") already marked recently. Please wait." << endl;
                this_thread::sleep_for(chrono::seconds(3)); // Give user time to read
                return;
            }
        }
        // Tentatively add to cooldown cache. It will be removed if no face detected.
        g_attendanceCooldownCache[uniqueId] = time(0);
    }

    VideoCapture cap(0);

    if (!cap.isOpened()) {
        cerr << "Error: Could not open video capture device. Check if webcam is connected and not in use." << endl;
        // Remove from cooldown cache if camera cannot be opened
        {
            lock_guard<mutex> lock(g_cooldownCacheMutex);
            g_attendanceCooldownCache.erase(uniqueId);
        }
        this_thread::sleep_for(chrono::seconds(5)); // Increased sleep for error
        return;
    }

    Mat frame;
    namedWindow("Face Detection", WINDOW_NORMAL);
    resizeWindow("Face Detection", 800, 600); // Slightly smaller window for better fit

    String face_cascade_path = "haarcascade_frontalface_default.xml";
    CascadeClassifier face_cascade;

    if (!face_cascade.load(face_cascade_path)) {
        cerr << "Error: Could not load face cascade. Make sure 'haarcascade_frontalface_default.xml' is in the same directory." << endl;
        destroyAllWindows();
        cap.release();
        // Remove from cooldown cache if cascade file cannot be loaded
        {
            lock_guard<mutex> lock(g_cooldownCacheMutex);
            g_attendanceCooldownCache.erase(uniqueId);
        }
        this_thread::sleep_for(chrono::seconds(5)); // Increased sleep for error
        return;
    }

    time_t start_time = time(0);
    bool faceSuccessfullyDetectedAndCaptured = false; // Flag to track successful face detection

    cout << "--- Capturing face for " << name << " (" << id << ") ---" << endl;
    cout << "Please look at the camera. Capturing for 4 seconds..." << endl; // Increased duration
    cout << "Press ESC to cancel." << endl;

    while (time(0) - start_time < 4) { // Capture for 4 seconds (increased from 3)
        cap >> frame;

        if (frame.empty()) {
            cerr << "Warning: Empty frame captured. Trying again..." << endl;
            this_thread::sleep_for(chrono::milliseconds(100));
            continue;
        }

        Mat gray;
        cvtColor(frame, gray, COLOR_BGR2GRAY);
        equalizeHist(gray, gray);

        vector<Rect> faces;
        face_cascade.detectMultiScale(gray, faces, 1.1, 3, 0, Size(30, 30));

        // Draw rectangles and save the first detected face (if any)
        if (!faces.empty()) {
            const Rect& face = faces[0];
            rectangle(frame, face, Scalar(0, 255, 0), 2); // Green rectangle
            string photoDir = "attendance_photos";
            createDirectoryIfNotExists(photoDir); // Ensure directory exists

            string photoPath = photoDir + "/" + name + "_" + id + ".jpg";
            imwrite(photoPath, frame(face)); // Save only the face ROI
            faceSuccessfullyDetectedAndCaptured = true; // Face detected and captured
            // No need to break, continue showing frames with detection for 4 seconds
        }

        // Add instructions and status to the video frame
        string instruction = "Capturing for " + to_string(4 - (time(0) - start_time)) + "s. Face: " + (faceSuccessfullyDetectedAndCaptured ? "Detected!" : "Searching...");
        putText(frame, instruction, Point(10, 30), FONT_HERSHEY_SIMPLEX, 0.7, Scalar(0, 255, 255), 2);
        putText(frame, "Press ESC to cancel.", Point(10, 60), FONT_HERSHEY_SIMPLEX, 0.5, Scalar(255, 255, 255), 1);

        imshow("Face Detection", frame);

        if (waitKey(1) == 27) { // ESC key pressed
            cout << "Face capture cancelled by user." << endl;
            faceSuccessfullyDetectedAndCaptured = false; // Ensure we don't mark attendance if cancelled
            break;
        }
    }

    destroyAllWindows();
    cap.release();

    // --- CRITICAL MODIFICATION: Check if face was successfully detected and captured ---
    if (!faceSuccessfullyDetectedAndCaptured) {
        cout << "? No face detected or captured during the process for " << name << " " << id << ". Attendance NOT marked." << endl;
        // Remove from cooldown cache if no face was successfully captured
        {
            lock_guard<mutex> lock(g_cooldownCacheMutex);
            g_attendanceCooldownCache.erase(uniqueId);
        }
        // No explicit sleep here; returnToMenu() will handle the pause
        return; // EXIT FUNCTION, DO NOT MARK ATTENDANCE
    }

    // If we reach here, a face was successfully detected and captured.
    cout << "? Face detected! Marking attendance..." << endl;

    // Format the log entry
    string currentLogEntry = getCurrentFormattedTime() + " PRESENT: " + name + " ID: " + id + " Photo: " + name + "_" + id + ".jpg";

    // --- Critical Fix: Write to specific log file synchronously ---
    ofstream specificLogFile(specificLogFileName, ios::app);
    if (specificLogFile.is_open()) {
        specificLogFile << currentLogEntry << endl;
        specificLogFile.close();
    }
    else {
        cerr << "Error: Could not open specific log file for writing: " << specificLogFileName << endl;
    }


    // --- Update DSA structures (protected by mutexes) ---

    // Update total attendance
    {
        lock_guard<mutex> lock(g_totalAttendanceMutex);
        totalAttendanceMap[uniqueId]++;
    }

    // Update recent attendance (std::list)
    {
        lock_guard<mutex> lock(g_recentAttendanceMutex);
        g_recentAttendance.push_front(currentLogEntry); // Add full entry for clarity
        if (g_recentAttendance.size() > MAX_RECENT_ATTENDEES) {
            g_recentAttendance.pop_back(); // Remove oldest if list is too long
        }
    }

    // Push to combined log queue (asynchronous logging)
    {
        lock_guard<mutex> lock(g_logQueueMutex);
        g_attendanceLogQueue.push(currentLogEntry);
    }
    g_logQueueCv.notify_one(); // Notify the logging thread that there's new data

    cout << "Attendance marked successfully for: " << name << " " << id << ". Logging asynchronously." << endl;
    // No explicit sleep here; returnToMenu() will handle the pause
}

// Function to display attendance from a specific log file for TODAY'S entries (using std::set)
void displayTodayAttendance(const string& logFileName) {
    clearScreen();
    cout << "\n--- Today's Attendance Report (" << getCurrentFormattedDate() << ") ---\n";
    cout << "-----------------------------------------------------\n";

    ifstream log(logFileName);
    if (!log.is_open()) {
        cerr << "Error opening attendance log file '" << logFileName << "'. It might not exist yet if no attendance has been marked." << endl;
        return;
    }

    set<string> todayAttendees; // Using a set for unique attendees today
    string line;
    string todayDate = getCurrentFormattedDate();

    while (getline(log, line)) {
        if (line.length() >= 10) {
            string logDate = line.substr(0, 10); // Extract "YYYY-MM-DD"
            if (logDate == todayDate) {
                // Parse name and ID from the log line (e.g., "YYYY-MM-DD HH:MM:SS PRESENT: Name ID: ID Photo:...")
                size_t presentPos = line.find("PRESENT: ");
                if (presentPos != string::npos) {
                    string attendeeInfo = line.substr(presentPos + 9); // +9 for "PRESENT: "
                    size_t photoPos = attendeeInfo.find(" Photo:");
                    if (photoPos != string::npos) {
                        attendeeInfo = attendeeInfo.substr(0, photoPos);
                    }
                    todayAttendees.insert(attendeeInfo); // Add to set
                }
            }
        }
    }
    log.close();

    if (todayAttendees.empty()) {
        cout << "\tNo attendance recorded for today (" << todayDate << ").\n";
    }
    else {
        cout << "\tUnique Attendees Today:\n";
        for (const string& attendee : todayAttendees) {
            cout << "\t- " << attendee << "\n";
        }
    }
    cout << "-----------------------------------------------------\n";
}

// Function to display attendance from a log file using BST for summary (Historical)
void displayAttendanceHistory(const string& logFileName) {
    clearScreen();
    cout << "\n--- Historical Attendance Log from " << logFileName << " ---\n";
    cout << "---------------------------------------------------------\n";

    ifstream log(logFileName);

    if (!log.is_open()) {
        cerr << "Error opening attendance log file '" << logFileName << "'. It might not exist yet if no attendance has been marked." << endl;
        return;
    }

    string line;
    AttendanceSummaryBST attendanceSummaryTree;

    while (getline(log, line)) {
        cout << "\t" << line << endl; // Display raw log entry

        // Parse date from the log line (assuming "YYYY-MM-DD HH:MM:SS PRESENT: Name ID: ID Photo:...")
        if (line.length() >= 10) { // Enough characters for "YYYY-MM-DD"
            string datePart = line.substr(0, 10); // Extract "YYYY-MM-DD"
            attendanceSummaryTree.insert(datePart); // Insert into our custom BST
        }
    }
    log.close();

    cout << "\n\n--- Attendance Summary by Date (from BST) ---\n";
    if (attendanceSummaryTree.isEmpty()) {
        cout << "\tNo attendance records found.\n";
    }
    else {
        attendanceSummaryTree.printSortedSummary(); // Print using BST's in-order traversal
    }
    cout << "---------------------------------------------------------\n";
}

// Display overall attendance statistics using Priority Queue
void displayOverallAttendanceStats(const unordered_map<string, int>& totalAttendanceMap, const string& category) {
    clearScreen();
    cout << "\n--- Overall " << category << " Attendance Statistics ---\n";
    cout << "--------------------------------------------------\n";

    // IMPORTANT: This now relies on totalAttendanceMap being populated by loadOverallAttendanceStats()
    // or by attendance marking within the current session.

    if (totalAttendanceMap.empty()) {
        cout << "\tNo overall attendance records yet for " << category << "s. Mark some attendance first!\n";
        return;
    }

    // Max-heap for Top 5 attendees (higher count first)
    priority_queue<pair<int, string>> topAttendees; // pair: {count, name_id}
    // Min-heap for Lowest 5 attendees (lower count first)
    priority_queue<pair<int, string>, vector<pair<int, string>>, greater<pair<int, string>>> lowestAttendees; // pair: {count, name_id}

    // Populate priority queues from the totalAttendanceMap
    {
        // No lock_guard needed here as this function is called from main thread after maps are loaded
        // or protected during updates in markAttendanceInternal.
        for (const auto& pair : totalAttendanceMap) {
            topAttendees.push({ pair.second, pair.first });
            lowestAttendees.push({ pair.second, pair.first });
        }
    }

    cout << "\n\tTop 5 " << category << "s by Attendance:\n";
    int count = 0;
    while (!topAttendees.empty() && count < 5) {
        cout << "\t" << count + 1 << ". " << topAttendees.top().second << " (Count: " << topAttendees.top().first << ")\n";
        topAttendees.pop();
        count++;
    }
    if (count == 0) cout << "\tN/A (No records to display)\n";


    cout << "\n\tLowest 5 " << category << "s by Attendance:\n";
    count = 0; // Reset count for lowest
    while (!lowestAttendees.empty() && count < 5) {
        cout << "\t" << count + 1 << ". " << lowestAttendees.top().second << " (Count: " << lowestAttendees.top().first << ")\n";
        lowestAttendees.pop();
        count++;
    }
    if (count == 0) cout << "\tN/A (No records to display)\n";

    cout << "--------------------------------------------------\n";
}

// Display recent attendance activity using std::list
void displayRecentAttendanceActivity() {
    clearScreen();
    cout << "\n--- Recent Attendance Activity ---\n";
    cout << " (Last " << MAX_RECENT_ATTENDEES << " marked entries during this session)\n";
    cout << "--------------------------------------\n";

    lock_guard<mutex> lock(g_recentAttendanceMutex); // Protect access to list

    if (g_recentAttendance.empty()) {
        cout << "\tNo recent attendance activity during this session.\n";
    }
    else {
        for (const string& entry : g_recentAttendance) {
            cout << "\t- " << entry << "\n";
        }
    }
    cout << "--------------------------------------\n";
}

// --- Function to check if a person was present on a specific date ---
void checkAttendanceOnSpecificDate() {
    clearScreen();
    cout << "\n--- Check Attendance on Specific Date ---\n";
    cout << "-------------------------------------------\n";

    string personName, personId, queryDateStr;
    string categoryChoice;
    bool isStudent = false;

    cout << "Is this person a (S)tudent or (T)eacher? (S/T): ";
    getline(cin, categoryChoice);
    if (categoryChoice == "S" || categoryChoice == "s") {
        isStudent = true;
    }
    else if (categoryChoice == "T" || categoryChoice == "t") {
        isStudent = false;
    }
    else {
        cout << "Invalid choice. Please enter S or T.\n";
        returnToMenu();
        return;
    }

    cout << "Enter Person's Name: ";
    getline(cin, personName);
    cout << "Enter Person's ID (Roll No for Student, Reg No for Teacher): ";
    getline(cin, personId);

    // IMPORTANT: For teachers, the unique ID in the log includes the shift (e.g., "T123_Shift1").
    // The user needs to provide that exact ID for a precise match.
    // If the user only provides "T123", it won't match "T123_Shift1".
    // It's generally better to capture the shift if checking teachers.
    string combinedIdForSearch = personId;
    if (!isStudent) {
        string shiftInput;
        cout << "Enter Teacher's Shift Number (if applicable, e.g., 1, 2, 3): ";
        getline(cin, shiftInput);
        if (!shiftInput.empty()) {
            combinedIdForSearch += "_Shift" + shiftInput;
        }
    }

    cout << "Enter Date to Check (YYYY-MM-DD): ";
    getline(cin, queryDateStr);

    if (personName.empty() || combinedIdForSearch.empty() || queryDateStr.empty()) {
        cout << "Name, ID, and Date cannot be empty.\n";
        returnToMenu();
        return;
    }

    string logFileName = isStudent ? "studentattendance_log.txt" : "teacherattendance_log.txt";
    ifstream logFile(logFileName);

    if (!logFile.is_open()) {
        cerr << "Error: Could not open log file '" << logFileName << "'. It might not exist yet.\n";
        returnToMenu();
        return;
    }

    string line;
    bool found = false;
    // How the person's info appears in the log after "PRESENT: "
    string searchPattern = personName + " ID: " + combinedIdForSearch;

    while (getline(logFile, line)) {
        // Check if the line starts with the query date
        if (line.rfind(queryDateStr, 0) == 0) { // rfind with 0 position checks for prefix
            // Check if the line contains the specific person's ID (and implicitly, their name)
            size_t presentInfoPos = line.find("PRESENT: ");
            if (presentInfoPos != string::npos) {
                string attendanceDetails = line.substr(presentInfoPos + 9); // After "PRESENT: "
                if (attendanceDetails.find(searchPattern) != string::npos) {
                    found = true;
                    cout << "\tFound attendance for " << personName << " (" << combinedIdForSearch << ") on " << queryDateStr << ".\n";
                    cout << "\tLog Entry: " << line << "\n";
                    break; // Found, no need to check further lines
                }
            }
        }
    }
    logFile.close();

    if (!found) {
        cout << "\t" << personName << " (" << combinedIdForSearch << ") was NOT recorded present on " << queryDateStr << ".\n";
    }
    cout << "-------------------------------------------\n";
    returnToMenu();
}


// --- Public Interface Functions (callable from main menu) ---

void markStudentAttendanceWrapper() { // Wrapper to simplify main menu call
    string name, rollNumber;
    cout << endl;
    cout << "\tEnter Student Name: ";
    getline(cin, name);
    cout << "\tEnter Roll Number: ";
    getline(cin, rollNumber);
    markAttendanceInternal(name, rollNumber, "studentattendance_log.txt", g_studentTotalAttendance);
    returnToMenu(); // Added returnToMenu() here
}

void markTeacherAttendanceWrapper() { // Wrapper to simplify main menu call
    string name, regNumber;
    int shiftNumber;
    cout << endl;
    cout << "\tEnter Teacher Name: ";
    getline(cin, name);
    cout << "\tEnter Registration Number: ";
    getline(cin, regNumber);

    cout << "\tEnter Shift Number: ";
    while (!(cin >> shiftNumber)) {
        cout << "\tInvalid input. Please enter a number for Shift Number: ";
        cin.clear();
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }
    cin.ignore(numeric_limits<streamsize>::max(), '\n'); // Clear buffer after reading shiftNumber

    // Combine RegNumber and Shift to form a unique ID for teachers in logs and maps
    markAttendanceInternal(name, regNumber + "_Shift" + to_string(shiftNumber), "teacherattendance_log.txt", g_teacherTotalAttendance);
    returnToMenu(); // Added returnToMenu() here
}

// --- NEW: Function to load overall attendance stats from log files at startup ---
void loadOverallAttendanceStats(const string& logFileName, unordered_map<string, int>& totalAttendanceMap) {
    ifstream logFile(logFileName);
    if (!logFile.is_open()) {
        // It's okay if the file doesn't exist yet, it just means no attendance has been marked.
        return;
    }

    string line;
    while (getline(logFile, line)) {
        // Example log format: "YYYY-MM-DD HH:MM:SS PRESENT: Name ID: ID Photo:..."
        size_t presentPos = line.find("PRESENT: ");
        if (presentPos != string::npos) {
            string attendeeInfo = line.substr(presentPos + 9); // Get "Name ID: ID Photo:..."

            // Find " ID: " to separate name from ID
            size_t idPos = attendeeInfo.find(" ID: ");
            if (idPos != string::npos) {
                string name = attendeeInfo.substr(0, idPos);
                string id_and_photo = attendeeInfo.substr(idPos + 5); // Get "ID Photo:..."

                size_t photoPos = id_and_photo.find(" Photo:");
                string id_only;
                if (photoPos != string::npos) {
                    id_only = id_and_photo.substr(0, photoPos);
                }
                else {
                    // Fallback if " Photo:" isn't found, assume rest is ID
                    id_only = id_and_photo;
                }

                string uniqueId = name + "_" + id_only;

                // Protect global map update with mutex
                lock_guard<mutex> lock(g_totalAttendanceMutex);
                totalAttendanceMap[uniqueId]++;
            }
        }
    }
    logFile.close();
}


// --- Main Program Flow ---
int main() {
    // Create necessary directories at startup
    createDirectoryIfNotExists("attendance_photos");

    // --- IMPORTANT: Load overall attendance stats from log files at program startup ---
    cout << "Loading historical student attendance data..." << endl;
    loadOverallAttendanceStats("studentattendance_log.txt", g_studentTotalAttendance);
    cout << "Loading historical teacher attendance data..." << endl;
    loadOverallAttendanceStats("teacherattendance_log.txt", g_teacherTotalAttendance);
    cout << "Data loading complete. Starting attendance system." << endl;
    this_thread::sleep_for(chrono::seconds(2)); // Give user time to read

    // Start the asynchronous log writer thread (for combined_attendance_log.txt)
    thread logWriter(logWriterFunction, "combined_attendance_log.txt");

    int choice;

    do {
        clearScreen();
        cout << endl;
        cout << endl;
        cout << "\t--- Attendance System Menu ---\n";
        cout << "\t1) Mark Student Attendance\n";
        cout << "\t2) Mark Teacher Attendance\n";
        cout << "\t3) View Today's Student Attendance\n";
        cout << "\t4) View Today's Teacher Attendance\n";
        cout << "\t5) View Historical Student Attendance (by Date)\n";
        cout << "\t6) View Historical Teacher Attendance (by Date)\n";
        cout << "\t7) View Overall Student Attendance Statistics\n";
        cout << "\t8) View Overall Teacher Attendance Statistics\n";
        cout << "\t9) View Recent Attendance Activity (during this session)\n";
        cout << "\t10) Check Attendance for a Specific Person on a Date\n";
        cout << "\t11) Exit\n";
        cout << "\tEnter your choice: ";
        cin >> choice;

        // Input validation for choice
        if (cin.fail() || choice < 1 || choice > 11) {
            cin.clear(); // Clear error flags
            cin.ignore(numeric_limits<streamsize>::max(), '\n'); // Discard invalid input
            cout << "\tInvalid choice. Please enter a number between 1 and 11.\n";
            returnToMenu();
            continue; // Skip the rest of the loop and show menu again
        }
        cin.ignore(numeric_limits<streamsize>::max(), '\n'); // Clear the input buffer after valid integer choice

        switch (choice) {
        case 1:
            markStudentAttendanceWrapper(); // returnToMenu() moved here
            break;
        case 2:
            markTeacherAttendanceWrapper(); // returnToMenu() moved here
            break;
        case 3:
            displayTodayAttendance("studentattendance_log.txt");
            returnToMenu();
            break;
        case 4:
            displayTodayAttendance("teacherattendance_log.txt");
            returnToMenu();
            break;
        case 5:
            displayAttendanceHistory("studentattendance_log.txt");
            returnToMenu();
            break;
        case 6:
            displayAttendanceHistory("teacherattendance_log.txt");
            returnToMenu();
            break;
        case 7:
            displayOverallAttendanceStats(g_studentTotalAttendance, "Student");
            returnToMenu();
            break;
        case 8:
            displayOverallAttendanceStats(g_teacherTotalAttendance, "Teacher");
            returnToMenu();
            break;
        case 9:
            displayRecentAttendanceActivity();
            returnToMenu();
            break;
        case 10:
            checkAttendanceOnSpecificDate();
            break;
        case 11:
            cout << "Exiting program.\n";
            break;
        default:
            cout << "An unexpected error occurred with your menu choice.\n"; // Fallback for unexpected cases
            returnToMenu();
            break;
        }
    } while (choice != 11);

    // Signal the logging thread to stop and wait for it to finish
    g_loggingThreadRunning = false;
    g_logQueueCv.notify_one(); // Notify to wake up the thread
    if (logWriter.joinable()) {
        logWriter.join(); // Wait for the logging thread to complete
    }

    return 0;
}