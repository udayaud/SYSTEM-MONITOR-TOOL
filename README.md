# SYSTEM-MONITOR-TOOL

#DAY1
#include <iostream>
#include <fstream>
#include <string>
#include <thread>
#include <chrono>

using namespace std;

double getCpuUsage() {
    static long prevIdle = 0, prevTotal = 0;
    ifstream file("/proc/stat");
    string cpu;
    long user, nice, system, idle, iowait, irq, softirq, steal;
    file >> cpu >> user >> nice >> system >> idle >> iowait >> irq >> softirq >> steal;
    long idleAll = idle + iowait;
    long total = user + nice + system + idle + iowait + irq + softirq + steal;
    long totalDiff = total - prevTotal;
    long idleDiff  = idleAll - prevIdle;
    prevTotal = total;
    prevIdle  = idleAll;
    if (totalDiff == 0) return 0.0;
    return (double)(totalDiff - idleDiff) / totalDiff * 100.0;
}

double getMemoryUsage() {
    ifstream file("/proc/meminfo");
    string key;
    long memTotal = 0, memAvailable = 0;
    while (file >> key) {
        if (key == "MemTotal:") file >> memTotal;
        else if (key == "MemAvailable:") { file >> memAvailable; break; }
        else file.ignore(numeric_limits<streamsize>::max(), '\n');
    }
    if (memTotal == 0) return 0.0;
    return (double)(memTotal - memAvailable) / memTotal * 100.0;
}

int main() {
    cout << "===== Simple System Monitor (Day 1) =====" << endl;
    while (true) {
        double cpu = getCpuUsage();
        double mem = getMemoryUsage();
        cout << "\rCPU Usage: " << cpu << "%   |   Memory Usage: " << mem << "%" << flush;
        this_thread::sleep_for(chrono::seconds(1));
    }
    return 0;
}




#DAY2&3

#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <dirent.h>
#include <unistd.h>
#include <algorithm>
#include <thread>
#include <chrono>
#include <iomanip>

using namespace std;

// Structure to hold process information
struct ProcessInfo {
    int pid;
    string name;
    double cpuUsage;
    double memUsage;
};

// Function to read total memory from /proc/meminfo
long getTotalMemory() {
    ifstream memFile("/proc/meminfo");
    string key;
    long memTotal = 0;
    while (memFile >> key) {
        if (key == "MemTotal:") {
            memFile >> memTotal;
            break;
        }
        memFile.ignore(numeric_limits<streamsize>::max(), '\n');
    }
    return memTotal;
}

// Function to collect all running processes and their CPU & memory usage
vector<ProcessInfo> getProcesses() {
    vector<ProcessInfo> processes;
    DIR* dir = opendir("/proc");
    if (!dir) return processes;

    struct dirent* entry;
    long totalMem = getTotalMemory();

    while ((entry = readdir(dir)) != nullptr) {
        if (!isdigit(entry->d_name[0])) continue;  // Only numeric folders are process IDs

        int pid = stoi(entry->d_name);
        string statPath = "/proc/" + string(entry->d_name) + "/stat";
        ifstream statFile(statPath);
        if (!statFile) continue;

        string name, skip;
        unsigned long utime = 0, stime = 0;
        statFile >> skip >> name;
        for (int i = 0; i < 11; i++) statFile >> skip; // skip unwanted data
        statFile >> utime >> stime;

        double cpuUsage = (utime + stime) / 100.0;

        // Read memory info
        string statusPath = "/proc/" + string(entry->d_name) + "/status";
        ifstream statusFile(statusPath);
        string key;
        long memKB = 0;
        while (statusFile >> key) {
            if (key == "VmRSS:") {
                statusFile >> memKB;
                break;
            }
            statusFile.ignore(numeric_limits<streamsize>::max(), '\n');
        }

        double memUsage = (double)memKB / totalMem * 100.0;
        processes.push_back({pid, name, cpuUsage, memUsage});
    }
    closedir(dir);
    return processes;
}

int main() {
    cout << "===== System Monitor (Day 2–3) =====" << endl;
    cout << "Displays running processes with CPU & Memory usage" << endl;
    cout << "Sorted by CPU usage (desc), then by Memory usage (desc)" << endl;
    cout << "Press Ctrl + C to stop the monitor" << endl;
    this_thread::sleep_for(chrono::seconds(2));

    while (true) {
        system("clear");
        vector<ProcessInfo> procs = getProcesses();

        // Sort first by CPU, then by Memory
        sort(procs.begin(), procs.end(), [](const ProcessInfo& a, const ProcessInfo& b) {
            if (a.cpuUsage == b.cpuUsage)
                return a.memUsage > b.memUsage;
            return a.cpuUsage > b.cpuUsage;
        });

        cout << left << setw(8) << "PID"
             << setw(20) << "Process"
             << setw(10) << "CPU%"
             << setw(10) << "MEM%" << endl;
        cout << string(50, '-') << endl;

        int count = 0;
        for (auto &p : procs) {
            if (count++ >= 10) break;
            cout << left << setw(8) << p.pid
                 << setw(20) << p.name.substr(0, 18)
                 << setw(10) << fixed << setprecision(2) << p.cpuUsage
                 << setw(10) << fixed << setprecision(2) << p.memUsage
                 << endl;
        }

        cout << "\n(Top 10 processes — Refreshing every 2 seconds...)" << endl;
        this_thread::sleep_for(chrono::seconds(2));
    }
    return 0;
}


#DAY4

#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <dirent.h>
#include <unistd.h>
#include <algorithm>
#include <thread>
#include <chrono>
#include <iomanip>
#include <ncurses.h>
#include <csignal>

using namespace std;

// Structure for process info
struct ProcessInfo {
    int pid;
    string name;
    double cpuUsage;
    double memUsage;
};

// Function to get total system memory
long getTotalMemory() {
    ifstream memFile("/proc/meminfo");
    string key;
    long memTotal = 0;
    while (memFile >> key) {
        if (key == "MemTotal:") {
            memFile >> memTotal;
            break;
        }
        memFile.ignore(numeric_limits<streamsize>::max(), '\n');
    }
    return memTotal;
}

// Function to read processes
vector<ProcessInfo> getProcesses() {
    vector<ProcessInfo> processes;
    DIR* dir = opendir("/proc");
    if (!dir) return processes;

    struct dirent* entry;
    long totalMem = getTotalMemory();

    while ((entry = readdir(dir)) != nullptr) {
        if (!isdigit(entry->d_name[0])) continue;
        int pid = stoi(entry->d_name);
        string statPath = "/proc/" + string(entry->d_name) + "/stat";
        ifstream statFile(statPath);
        if (!statFile) continue;

        string name, skip;
        unsigned long utime = 0, stime = 0;
        statFile >> skip >> name;
        for (int i = 0; i < 11; i++) statFile >> skip;
        statFile >> utime >> stime;

        double cpuUsage = (utime + stime) / 100.0;

        // Memory info
        string statusPath = "/proc/" + string(entry->d_name) + "/status";
        ifstream statusFile(statusPath);
        string key;
        long memKB = 0;
        while (statusFile >> key) {
            if (key == "VmRSS:") {
                statusFile >> memKB;
                break;
            }
            statusFile.ignore(numeric_limits<streamsize>::max(), '\n');
        }

        double memUsage = (double)memKB / totalMem * 100.0;
        processes.push_back({pid, name, cpuUsage, memUsage});
    }

    closedir(dir);
    return processes;
}

// Function to draw UI
void drawUI(const vector<ProcessInfo>& procs) {
    clear();
    mvprintw(0, 0, "===== System Monitor (Day 4) =====");
    mvprintw(1, 0, "Press 'q' to quit | Press 'k' to kill process");
    mvprintw(3, 0, "PID     %-20s  CPU%%   MEM%%", "Process Name");
    mvhline(4, 0, '-', 50);

    int row = 5;
    for (int i = 0; i < (int)procs.size() && i < 10; ++i) {
        const auto& p = procs[i];

        // Color based on CPU usage
        if (p.cpuUsage > 10.0)
            attron(COLOR_PAIR(3)); // red
        else if (p.cpuUsage > 5.0)
            attron(COLOR_PAIR(2)); // yellow
        else
            attron(COLOR_PAIR(1)); // green

        mvprintw(row++, 0, "%-7d %-20s %-7.2f %-7.2f",
                 p.pid, p.name.substr(0, 18).c_str(),
                 p.cpuUsage, p.memUsage);

        attroff(COLOR_PAIR(1));
        attroff(COLOR_PAIR(2));
        attroff(COLOR_PAIR(3));
    }
    refresh();
}

int main() {
    // Initialize ncurses
    initscr();
    noecho();
    cbreak();
    nodelay(stdscr, TRUE);
    start_color();
    init_pair(1, COLOR_GREEN, COLOR_BLACK);
    init_pair(2, COLOR_YELLOW, COLOR_BLACK);
    init_pair(3, COLOR_RED, COLOR_BLACK);

    bool running = true;

    while (running) {
        vector<ProcessInfo> procs = getProcesses();

        // Sort by CPU, then by Memory
        sort(procs.begin(), procs.end(), [](const ProcessInfo& a, const ProcessInfo& b) {
            if (a.cpuUsage == b.cpuUsage)
                return a.memUsage > b.memUsage;
            return a.cpuUsage > b.cpuUsage;
        });

        drawUI(procs);

        int ch = getch();
        if (ch == 'q') running = false;
        else if (ch == 'k') {
            echo();
            nodelay(stdscr, FALSE);
            mvprintw(17, 0, "Enter PID to kill: ");
            int pid;
            scanw("%d", &pid);
            if (kill(pid, SIGKILL) == 0)
                mvprintw(18, 0, "Process %d killed successfully.", pid);
            else
                mvprintw(18, 0, "Failed to kill %d (try sudo?).", pid);
            noecho();
            nodelay(stdscr, TRUE);
        }

        refresh();
        this_thread::sleep_for(chrono::seconds(2));
    }

    endwin();
    return 0;
}



#DAY5


#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <dirent.h>
#include <unistd.h>
#include <algorithm>
#include <thread>
#include <chrono>
#include <iomanip>
#include <ncurses.h>
#include <ctime>
#include <csignal>

using namespace std;

// Structure to hold process info
struct ProcessInfo {
    int pid;
    string name;
    double cpuUsage;
    double memUsage;
};

// Function to read total memory
long getTotalMemory() {
    ifstream memFile("/proc/meminfo");
    string key;
    long memTotal = 0;
    while (memFile >> key) {
        if (key == "MemTotal:") {
            memFile >> memTotal;
            break;
        }
        memFile.ignore(numeric_limits<streamsize>::max(), '\n');
    }
    return memTotal;
}

// Function to get all running processes
vector<ProcessInfo> getProcesses() {
    vector<ProcessInfo> processes;
    DIR* dir = opendir("/proc");
    if (!dir) return processes;

    struct dirent* entry;
    long totalMem = getTotalMemory();

    while ((entry = readdir(dir)) != nullptr) {
        if (!isdigit(entry->d_name[0])) continue;

        int pid = stoi(entry->d_name);
        string statPath = "/proc/" + string(entry->d_name) + "/stat";
        ifstream statFile(statPath);
        if (!statFile) continue;

        string name;
        string skip;
        unsigned long utime = 0, stime = 0;

        statFile >> skip >> name;
        for (int i = 0; i < 11; i++) statFile >> skip;
        statFile >> utime >> stime;

        double cpuUsage = (utime + stime) / 100.0;

        // Memory usage
        string statusPath = "/proc/" + string(entry->d_name) + "/status";
        ifstream statusFile(statusPath);
        string key;
        long memKB = 0;
        while (statusFile >> key) {
            if (key == "VmRSS:") {
                statusFile >> memKB;
                break;
            }
            statusFile.ignore(numeric_limits<streamsize>::max(), '\n');
        }

        double memUsage = (double)memKB / totalMem * 100.0;
        processes.push_back({pid, name, cpuUsage, memUsage});
    }

    closedir(dir);
    return processes;
}

int main() {
    // Start ncurses mode
    initscr();
    noecho();
    cbreak();
    nodelay(stdscr, TRUE); // non-blocking key input
    start_color();

    // Define color pairs
    init_pair(1, COLOR_GREEN, COLOR_BLACK);
    init_pair(2, COLOR_YELLOW, COLOR_BLACK);
    init_pair(3, COLOR_RED, COLOR_BLACK);

    bool running = true;

    while (running) {
        system("clear"); // refresh screen content

        vector<ProcessInfo> procs = getProcesses();

        // Sort by CPU usage first, then by Memory usage
        sort(procs.begin(), procs.end(), [](const ProcessInfo& a, const ProcessInfo& b) {
            if (a.cpuUsage == b.cpuUsage)
                return a.memUsage > b.memUsage;
            return a.cpuUsage > b.cpuUsage;
        });

        // Title and header
        mvprintw(0, 0, "===== Simple System Monitor (Day 5 - Real-Time + Kill Feature) =====");
        mvprintw(1, 0, "Press 'q' to quit | Press 'k' to kill a process | Auto-refresh every 2s");
        mvprintw(3, 0, "PID     %-20s  CPU%%   MEM%%", "Process Name");
        mvhline(4, 0, '-', 50);

        // Display top 10 processes
        int row = 5;
        for (int i = 0; i < (int)procs.size() && i < 10; ++i) {
            const auto& p = procs[i];

            if (p.cpuUsage > 10.0)
                attron(COLOR_PAIR(3));
            else if (p.cpuUsage > 5.0)
                attron(COLOR_PAIR(2));
            else
                attron(COLOR_PAIR(1));

            mvprintw(row++, 0, "%-7d %-20s %-7.2f %-7.2f",
                     p.pid, p.name.substr(0, 18).c_str(),
                     p.cpuUsage, p.memUsage);

            attroff(COLOR_PAIR(1));
            attroff(COLOR_PAIR(2));
            attroff(COLOR_PAIR(3));
        }

        // Show timestamp of last update
        time_t now = time(0);
        string timeStr = ctime(&now);
        timeStr.pop_back(); // remove newline
        mvprintw(17, 0, "Last Updated: %s", timeStr.c_str());
        mvprintw(18, 0, "(Top 10 processes sorted by CPU and Memory)");

        // Check for user input
        int ch = getch();
        if (ch == 'q' || ch == 'Q') {
            running = false;
        } 
        else if (ch == 'k' || ch == 'K') {
            echo();
            nodelay(stdscr, FALSE);
            mvprintw(20, 0, "Enter PID to kill: ");
            int pid;
            scanw("%d", &pid);

            if (kill(pid, SIGKILL) == 0)
                mvprintw(21, 0, "Process %d killed successfully.", pid);
            else
                mvprintw(21, 0, "Failed to kill %d. (Try using sudo)", pid);

            noecho();
            nodelay(stdscr, TRUE);
        }

        refresh();
        this_thread::sleep_for(chrono::seconds(2)); // refresh every 2 seconds
    }

    endwin();
    return 0;
}
