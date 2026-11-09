# Sales-Data-Summary-Tool
A C++ command-line tool that reads monthly sales data from a CSV file and reports total revenue, average unit price, best and worst months, month-to-month growth, and a console bar chart.
// Sales Data Summary Tool
// Reads monthly sales from a CSV file and reports totals, averages,
// best/worst months, growth, and a simple console bar chart.

#include <algorithm>
#include <fstream>
#include <iomanip>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

struct Record {
    std::string month;
    double revenue;
    int units;
};

// Remove spaces and '\r' from both ends of a string
std::string trim(const std::string& s) {
    size_t start = s.find_first_not_of(" \t\r\n");
    if (start == std::string::npos) return "";
    size_t end = s.find_last_not_of(" \t\r\n");
    return s.substr(start, end - start + 1);
}

// Load records from CSV (expects header: Month,Revenue,Units)
bool loadCSV(const std::string& filename, std::vector<Record>& data) {
    std::ifstream file(filename);
    if (!file) return false;

    std::string line;
    std::getline(file, line);  // skip header

    int lineNo = 1;
    while (std::getline(file, line)) {
        ++lineNo;
        if (trim(line).empty()) continue;

        std::stringstream ss(line);
        std::string month, rev, units;
        std::getline(ss, month, ',');
        std::getline(ss, rev, ',');
        std::getline(ss, units, ',');

        try {
            data.push_back({trim(month), std::stod(trim(rev)), std::stoi(trim(units))});
        } catch (...) {
            std::cerr << "Warning: skipping invalid line " << lineNo << ": " << line << "\n";
        }
    }
    return true;
}

double totalRevenue(const std::vector<Record>& d) {
    double sum = 0;
    for (const auto& r : d) sum += r.revenue;
    return sum;
}

int totalUnits(const std::vector<Record>& d) {
    int sum = 0;
    for (const auto& r : d) sum += r.units;
    return sum;
}

const Record& bestMonth(const std::vector<Record>& d) {
    return *std::max_element(d.begin(), d.end(),
        [](const Record& a, const Record& b) { return a.revenue < b.revenue; });
}

const Record& worstMonth(const std::vector<Record>& d) {
    return *std::min_element(d.begin(), d.end(),
        [](const Record& a, const Record& b) { return a.revenue < b.revenue; });
}

void printBarChart(std::ostream& out, const std::vector<Record>& d, double maxRevenue) {
    const int maxBar = 40;
    for (const auto& r : d) {
        int len = static_cast<int>(r.revenue / maxRevenue * maxBar);
        out << std::left << std::setw(12) << r.month << "| "
            << std::string(len, '#') << " " << std::fixed << std::setprecision(0)
            << r.revenue << "\n";
    }
}

void printReport(std::ostream& out, const std::vector<Record>& d) {
    double total = totalRevenue(d);
    double avg = total / d.size();
    const Record& best = bestMonth(d);
    const Record& worst = worstMonth(d);

    out << std::fixed << std::setprecision(2);
    out << "========== SALES SUMMARY ==========\n";
    out << "Months analysed     : " << d.size() << "\n";
    out << "Total revenue       : " << total << "\n";
    out << "Total units sold    : " << totalUnits(d) << "\n";
    out << "Average revenue     : " << avg << " per month\n";
    out << "Average unit price  : " << total / totalUnits(d) << "\n\n";

    out << "Best month          : " << best.month << " (" << best.revenue << ")\n";
    out << "Worst month         : " << worst.month << " (" << worst.revenue << ")\n";
    out << "Best vs worst gap   : " << best.revenue - worst.revenue << "\n\n";

    out << "Months above average:";
    for (const auto& r : d)
        if (r.revenue > avg) out << " " << r.month;
    out << "\n\n";

    out << "Month-to-month growth:\n";
    for (size_t i = 1; i < d.size(); ++i) {
        double change = (d[i].revenue - d[i - 1].revenue) / d[i - 1].revenue * 100.0;
        out << "  " << std::left << std::setw(12) << d[i].month
            << std::right << std::setw(8) << std::showpos << change
            << std::noshowpos << " %\n";
    }

    out << "\nRevenue chart:\n";
    printBarChart(out, d, best.revenue);
    out << "===================================\n";
}

int main(int argc, char* argv[]) {
    std::string filename = (argc > 1) ? argv[1] : "sales.csv";

    std::vector<Record> data;
    if (!loadCSV(filename, data)) {
        std::cerr << "Error: could not open '" << filename << "'\n";
        return 1;
    }
    if (data.empty()) {
        std::cerr << "Error: no valid data found in file.\n";
        return 1;
    }

    printReport(std::cout, data);

    std::ofstream report("report.txt");
    if (report) {
        printReport(report, data);
        std::cout << "Report saved to report.txt\n";
    }
    return 0;
}
