import requests
from urllib.parse import urljoin, urlparse
from bs4 import BeautifulSoup
import threading
from queue import Queue
import time
import json
import tkinter as tk
from tkinter import ttk, scrolledtext, messagebox
from collections import defaultdict
import re

class WebCrawler:
    def __init__(self, base_url, max_threads=5, max_urls=100):
        self.base_url = base_url
        self.domain = urlparse(base_url).netloc
        self.visited_urls = set()
        self.urls_to_visit = Queue()
        self.urls_to_visit.put(base_url)
        self.max_threads = max_threads
        self.max_urls = max_urls
        self.endpoints = defaultdict(list)
        self.lock = threading.Lock()
        self.crawling = False
        
    def is_same_domain(self, url):
        return urlparse(url).netloc == self.domain
        
    def get_links(self, url):
        try:
            headers = {
                'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
            }
            response = requests.get(url, headers=headers, timeout=5)
            soup = BeautifulSoup(response.content, 'html.parser')
            
            # Extract all links
            links = []
            for link in soup.find_all('a', href=True):
                absolute_url = urljoin(url, link['href'])
                if self.is_same_domain(absolute_url):
                    links.append(absolute_url)
            
            # Extract forms and their actions as endpoints
            forms = []
            for form in soup.find_all('form'):
                if form.get('action'):
                    action_url = urljoin(url, form.get('action'))
                    method = form.get('method', 'GET').upper()
                    inputs = []
                    
                    for input_tag in form.find_all('input'):
                        inputs.append({
                            'name': input_tag.get('name', ''),
                            'type': input_tag.get('type', 'text'),
                            'value': input_tag.get('value', '')
                        })
                    
                    forms.append({
                        'action': action_url,
                        'method': method,
                        'inputs': inputs
                    })
            
            return links, forms, response.status_code
        except Exception as e:
            return [], [], None
        
    def crawl_worker(self, results_callback):
        while self.crawling and not self.urls_to_visit.empty():
            if len(self.visited_urls) >= self.max_urls:
                break
                
            try:
                current_url = self.urls_to_visit.get(timeout=1)
                
                with self.lock:
                    if current_url in self.visited_urls:
                        self.urls_to_visit.task_done()
                        continue
                    self.visited_urls.add(current_url)
                
                links, forms, status_code = self.get_links(current_url)
                
                # Add newly discovered URLs to the queue
                for link in links:
                    with self.lock:
                        if link not in self.visited_urls and not any(link in q_item for q_item in list(self.urls_to_visit.queue)):
                            self.urls_to_visit.put(link)
                
                # Store endpoints
                with self.lock:
                    if forms:
                        self.endpoints[current_url] = forms
                
                # Callback to update UI
                if results_callback:
                    results_callback({
                        'url': current_url,
                        'links_found': len(links),
                        'forms_found': len(forms),
                        'status': status_code,
                        'total_visited': len(self.visited_urls),
                        'queue_size': self.urls_to_visit.qsize()
                    })
                
                self.urls_to_visit.task_done()
                time.sleep(0.1)  # Be polite
                
            except Exception as e:
                self.urls_to_visit.task_done()
                continue
    
    def start_crawling(self, results_callback=None):
        self.crawling = True
        
        # Create and start worker threads
        threads = []
        for _ in range(self.max_threads):
            thread = threading.Thread(target=self.crawl_worker, args=(results_callback,))
            thread.daemon = True
            thread.start()
            threads.append(thread)
        
        # Wait for all URLs to be processed or max URLs reached
        self.urls_to_visit.join()
        
        # Wait for all threads to complete
        self.crawling = False
        for thread in threads:
            thread.join(timeout=1)
    
    def stop_crawling(self):
        self.crawling = False
        
    def get_results(self):
        return {
            'visited_urls': list(self.visited_urls),
            'endpoints': dict(self.endpoints),
            'total_urls_visited': len(self.visited_urls),
            'total_endpoints_found': sum(len(forms) for forms in self.endpoints.values())
        }

class CrawlerGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Web Crawler and Endpoint Discovery")
        self.root.geometry("900x700")
        self.crawler = None
        
        # Create main frame
        main_frame = ttk.Frame(root, padding="10")
        main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        # Input section
        input_frame = ttk.LabelFrame(main_frame, text="Crawler Settings", padding="5")
        input_frame.grid(row=0, column=0, columnspan=2, sticky=(tk.W, tk.E), pady=5)
        
        ttk.Label(input_frame, text="Start URL:").grid(row=0, column=0, sticky=tk.W, pady=2)
        self.url_entry = ttk.Entry(input_frame, width=60)
        self.url_entry.grid(row=0, column=1, columnspan=2, sticky=(tk.W, tk.E), pady=2, padx=5)
        self.url_entry.insert(0, "https://httpbin.org/")  # Example URL
        
        ttk.Label(input_frame, text="Max Threads:").grid(row=1, column=0, sticky=tk.W, pady=2)
        self.threads_entry = ttk.Entry(input_frame, width=10)
        self.threads_entry.grid(row=1, column=1, sticky=tk.W, pady=2, padx=5)
        self.threads_entry.insert(0, "5")
        
        ttk.Label(input_frame, text="Max URLs:").grid(row=1, column=2, sticky=tk.W, pady=2, padx=10)
        self.max_urls_entry = ttk.Entry(input_frame, width=10)
        self.max_urls_entry.grid(row=1, column=3, sticky=tk.W, pady=2, padx=5)
        self.max_urls_entry.insert(0, "50")
        
        # Buttons
        self.start_btn = ttk.Button(input_frame, text="Start Crawling", command=self.start_crawling)
        self.start_btn.grid(row=2, column=0, pady=10, padx=5)
        
        self.stop_btn = ttk.Button(input_frame, text="Stop Crawling", command=self.stop_crawling, state=tk.DISABLED)
        self.stop_btn.grid(row=2, column=1, pady=10, padx=5)
        
        # Progress section
        progress_frame = ttk.LabelFrame(main_frame, text="Progress", padding="5")
        progress_frame.grid(row=1, column=0, sticky=(tk.W, tk.E, tk.N, tk.S), pady=5)
        
        self.progress_text = scrolledtext.ScrolledText(progress_frame, width=85, height=15)
        self.progress_text.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        # Results section
        results_frame = ttk.LabelFrame(main_frame, text="Discovered Endpoints", padding="5")
        results_frame.grid(row=2, column=0, sticky=(tk.W, tk.E, tk.N, tk.S), pady=5)
        
        # Treeview for endpoints
        columns = ('page_url', 'endpoint', 'method', 'parameters')
        self.tree = ttk.Treeview(results_frame, columns=columns, show='headings', height=10)
        
        self.tree.heading('page_url', text='Page URL')
        self.tree.heading('endpoint', text='Endpoint')
        self.tree.heading('method', text='Method')
        self.tree.heading('parameters', text='Parameters')
        
        self.tree.column('page_url', width=250)
        self.tree.column('endpoint', width=200)
        self.tree.column('method', width=80)
        self.tree.column('parameters', width=300)
        
        self.tree.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        # Scrollbar for treeview
        scrollbar = ttk.Scrollbar(results_frame, orient=tk.VERTICAL, command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        scrollbar.grid(row=0, column=1, sticky=(tk.N, tk.S))
        
        # Status bar
        self.status_var = tk.StringVar()
        self.status_var.set("Ready to start crawling")
        status_bar = ttk.Label(main_frame, textvariable=self.status_var, relief=tk.SUNKEN)
        status_bar.grid(row=3, column=0, sticky=(tk.W, tk.E), pady=5)
        
        # Configure grid weights
        root.columnconfigure(0, weight=1)
        root.rowconfigure(0, weight=1)
        main_frame.columnconfigure(0, weight=1)
        main_frame.rowconfigure(1, weight=1)
        main_frame.rowconfigure(2, weight=1)
        progress_frame.columnconfigure(0, weight=1)
        progress_frame.rowconfigure(0, weight=1)
        results_frame.columnconfigure(0, weight=1)
        results_frame.rowconfigure(0, weight=1)
        
    def log_message(self, message):
        self.progress_text.insert(tk.END, message + "\n")
        self.progress_text.see(tk.END)
        self.root.update_idletasks()
        
    def update_status(self, data):
        message = f"Visited: {data['total_visited']} | Queue: {data['queue_size']} | Links: {data['links_found']} | Forms: {data['forms_found']} | Status: {data['status']}"
        self.status_var.set(message)
        self.log_message(f"Crawled: {data['url']}")
        
    def start_crawling(self):
        url = self.url_entry.get().strip()
        if not url:
            messagebox.showerror("Error", "Please enter a valid URL")
            return
            
        try:
            max_threads = int(self.threads_entry.get())
            max_urls = int(self.max_urls_entry.get())
        except ValueError:
            messagebox.showerror("Error", "Please enter valid numbers for threads and max URLs")
            return
            
        # Clear previous results
        self.progress_text.delete(1.0, tk.END)
        for item in self.tree.get_children():
            self.tree.delete(item)
            
        # Initialize crawler
        self.crawler = WebCrawler(url, max_threads, max_urls)
        
        # Update UI state
        self.start_btn.config(state=tk.DISABLED)
        self.stop_btn.config(state=tk.NORMAL)
        
        # Start crawling in a separate thread to keep UI responsive
        def crawl_thread():
            self.log_message(f"Starting crawl of {url}")
            self.log_message(f"Max threads: {max_threads}, Max URLs: {max_urls}")
            self.log_message("=" * 50)
            
            self.crawler.start_crawling(self.update_status)
            
            # Update UI when done
            self.root.after(0, self.crawling_finished)
            
        threading.Thread(target=crawl_thread, daemon=True).start()
        
    def stop_crawling(self):
        if self.crawler:
            self.crawler.stop_crawling()
            self.log_message("Crawling stopped by user")
            
    def crawling_finished(self):
        results = self.crawler.get_results()
        
        self.log_message("=" * 50)
        self.log_message(f"Crawling completed!")
        self.log_message(f"Total URLs visited: {results['total_urls_visited']}")
        self.log_message(f"Total endpoints discovered: {results['total_endpoints_found']}")
        
        # Display endpoints in treeview
        for page_url, forms in results['endpoints'].items():
            for form in forms:
                params = ", ".join([f"{inp['name']} ({inp['type']})" for inp in form['inputs'] if inp['name']])
                self.tree.insert('', tk.END, values=(
                    page_url[:40] + "..." if len(page_url) > 40 else page_url,
                    form['action'],
                    form['method'],
                    params
                ))
        
        # Update UI state
        self.start_btn.config(state=tk.NORMAL)
        self.stop_btn.config(state=tk.DISABLED)
        self.status_var.set(f"Crawling completed. Visited {results['total_urls_visited']} URLs, found {results['total_endpoints_found']} endpoints.")

if __name__ == "__main__":
    root = tk.Tk()
    app = CrawlerGUI(root)
    root.mainloop()
