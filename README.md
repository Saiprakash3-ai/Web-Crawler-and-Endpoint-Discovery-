# Web-Crawler-and-Endpoint-Discovery- 

# How to Use This Web Crawler

Run the script, and a GUI window will open.

Enter the URL you want to crawl (e.g., https://example.com/).

Set the number of threads (parallel requests) and maximum URLs to crawl.

Click "Start Crawling" to begin the process.

The progress text area will show visited URLs in real-time.

Discovered endpoints (forms) will be displayed in the table with their methods and parameters.

Click "Stop Crawling" to halt the process at any time.

# Features

Multi-threaded crawling for efficiency

Endpoint discovery through form analysis

Real-time progress updates

Clean GUI interface

Respectful crawling with delays between requests

Results displayed in an organized table

# Requirements

Install the required libraries with:
text

    pip install requests beautifulsoup4

# Notes

The crawler respects robots.txt by default (you can enhance this)

It only follows links within the same domain

Forms are analyzed to discover endpoints with their methods and parameters

The tool is designed to be ethical - only crawl websites you have permission to test
