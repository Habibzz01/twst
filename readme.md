#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# GitHub Ultimate Terminal Assistant (GUTA)
# Developer: Xbibz Official - MR. Nexo444
# Version: 2.1.0
# License: MIT

import os
import sys
import time
import json
import random
import shutil
import platform
import subprocess
import webbrowser
import base64
from datetime import datetime
from getpass import getpass
from typing import Dict, List, Optional, Union
import requests
from bs4 import BeautifulSoup
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException, WebDriverException
import pyfiglet
from colorama import init, Fore, Style
from tqdm import tqdm
from cryptography.fernet import Fernet
import readline

# Initialize colorama
init(autoreset=True)

# Basic configuration
CONFIG_FILE = os.path.expanduser("~/.guta_config.json")
TOKEN_FILE = os.path.expanduser("~/.guta_tokens.json")
LOG_FILE = os.path.expanduser("~/.guta_log.txt")
REPO_URL = "https://raw.githubusercontent.com/xbibz-official/guta/main/guta.py"
UPDATE_INTERVAL = 86400  # 24 hours in seconds

# Colors for display
COLORS = {
    "success": Fore.GREEN,
    "error": Fore.RED,
    "warning": Fore.YELLOW,
    "info": Fore.CYAN,
    "debug": Fore.MAGENTA,
    "banner": Fore.LIGHTMAGENTA_EX,
    "menu": Fore.LIGHTBLUE_EX,
    "input": Fore.LIGHTWHITE_EX,
    "neon": [Fore.LIGHTCYAN_EX, Fore.LIGHTMAGENTA_EX, Fore.LIGHTYELLOW_EX, Fore.LIGHTGREEN_EX]
}

# Loading animation
LOADING_FRAMES = [
    "⣾", "⣽", "⣻", "⢿", "⡿", "⣟", "⣯", "⣷"
]

class GUTA:
    def __init__(self):
        self.config = self.load_config()
        self.github_tokens = self.load_tokens()
        self.current_token = None
        self.driver = None
        self.session = requests.Session()
        self.session.headers.update({"User-Agent": "GUTA/2.1 (Xbibz Official)"})
        self.fernet = Fernet(self.generate_or_get_key())
        self.last_update_check = 0
        self.command_history = []
        
        # Setup readline for command history
        readline.parse_and_bind("tab: complete")
        readline.set_completer(self.completer)
        
        # Check for updates on first run
        self.check_update(force_notification=False)
    
    def generate_or_get_key(self) -> bytes:
        """Generate or read encryption key"""
        key_file = os.path.expanduser("~/.guta_key.key")
        if os.path.exists(key_file):
            with open(key_file, "rb") as f:
                return f.read()
        else:
            key = Fernet.generate_key()
            with open(key_file, "wb") as f:
                f.write(key)
            return key
    
    def completer(self, text: str, state: int) -> Optional[str]:
        """Auto-completion for commands"""
        options = [cmd for cmd in COMMANDS if cmd.startswith(text)]
        if state < len(options):
            return options[state]
        return None
    
    def load_config(self) -> Dict:
        """Load or create new config file"""
        default_config = {
            "auto_update": True,
            "theme": "neon",
            "animation_speed": 1,
            "last_repo": "",
            "github_username": "",
            "show_banner": True
        }
        
        try:
            if os.path.exists(CONFIG_FILE):
                with open(CONFIG_FILE, "r") as f:
                    return json.load(f)
            else:
                with open(CONFIG_FILE, "w") as f:
                    json.dump(default_config, f, indent=4)
                return default_config
        except Exception as e:
            self.log_error(f"Failed to load config: {str(e)}")
            return default_config
    
    def save_config(self) -> None:
        """Save config to file"""
        try:
            with open(CONFIG_FILE, "w") as f:
                json.dump(self.config, f, indent=4)
        except Exception as e:
            self.log_error(f"Failed to save config: {str(e)}")
    
    def load_tokens(self) -> Dict:
        """Load or create new token file"""
        default_tokens = {
            "tokens": {},
            "active": None
        }
        
        try:
            if os.path.exists(TOKEN_FILE):
                with open(TOKEN_FILE, "r") as f:
                    encrypted = f.read()
                    decrypted = self.fernet.decrypt(encrypted.encode()).decode()
                    return json.loads(decrypted)
            else:
                with open(TOKEN_FILE, "w") as f:
                    encrypted = self.fernet.encrypt(json.dumps(default_tokens).encode()).decode()
                    f.write(encrypted)
                return default_tokens
        except Exception as e:
            self.log_error(f"Failed to load tokens: {str(e)}")
            return default_tokens
    
    def save_tokens(self) -> None:
        """Save tokens to file"""
        try:
            with open(TOKEN_FILE, "w") as f:
                encrypted = self.fernet.encrypt(json.dumps(self.github_tokens).encode()).decode()
                f.write(encrypted)
        except Exception as e:
            self.log_error(f"Failed to save tokens: {str(e)}")
    
    def log_error(self, message: str) -> None:
        """Log error to file"""
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        log_entry = f"[{timestamp}] ERROR: {message}\n"
        
        try:
            with open(LOG_FILE, "a") as f:
                f.write(log_entry)
        except:
            pass
    
    def animate_text(self, text: str, color: str = None, delay: float = 0.05) -> None:
        """Typewriter text animation"""
        if color is None:
            color = random.choice(COLORS["neon"])
            
        for char in text:
            sys.stdout.write(color + char)
            sys.stdout.flush()
            time.sleep(delay * self.config["animation_speed"])
        print()
    
    def loading_animation(self, duration: float = 2.0, text: str = "Loading") -> None:
        """Loading animation"""
        end_time = time.time() + duration
        frame_idx = 0
        
        while time.time() < end_time:
            frame = LOADING_FRAMES[frame_idx % len(LOADING_FRAMES)]
            color = COLORS["neon"][frame_idx % len(COLORS["neon"])]
            sys.stdout.write(f"\r{color}{frame} {text}...{Style.RESET_ALL}")
            sys.stdout.flush()
            frame_idx += 1
            time.sleep(0.1)
        
        sys.stdout.write("\r" + " " * (len(text) + 10) + "\r")
        sys.stdout.flush()
    
    def clear_screen(self) -> None:
        """Clear terminal screen"""
        os.system("cls" if os.name == "nt" else "clear")
    
    def show_banner(self) -> None:
        """Display ASCII art banner"""
        if not self.config["show_banner"]:
            return
            
        self.clear_screen()
        banner_text = pyfiglet.figlet_format("X = Github", font="slant")
        lines = banner_text.split("\n")
        
        for line in lines:
            color = random.choice(COLORS["neon"])
            self.animate_text(line, color, 0.005)
        
        version_text = f"GitHub Ultimate Terminal Assistant v2.1.0 | By Xbibz Official - MR. Nexo444"
        print(COLORS["banner"] + "═" * (len(version_text) + 10))
        self.animate_text(version_text, COLORS["banner"], 0.02)
        print(COLORS["banner"] + "═" * (len(version_text) + 10) + Style.RESET_ALL)
        print()
        
        # Display system info
        self.show_system_info()
        print()
    
    def show_system_info(self) -> None:
        """Display system information"""
        info = [
            f"OS: {platform.system()} {platform.release()}",
            f"Python: {platform.python_version()}",
            f"Terminal: {os.getenv('TERM', 'Unknown')}",
            f"Device: {platform.node()}",
            f"CPU: {platform.processor() or 'Unknown'}"
        ]
        
        for i, line in enumerate(info):
            color = COLORS["neon"][i % len(COLORS["neon"])]
            print(f"{color}• {line}{Style.RESET_ALL}")
    
    def check_dependencies(self) -> None:
        """Check and install dependencies"""
        required = {
            "requests": "requests",
            "beautifulsoup4": "bs4",
            "selenium": "selenium",
            "pyfiglet": "pyfiglet",
            "colorama": "colorama",
            "tqdm": "tqdm",
            "cryptography": "cryptography"
        }
        
        missing = []
        
        for pkg, module in required.items():
            try:
                __import__(module)
            except ImportError:
                missing.append(pkg)
        
        if missing:
            print(COLORS["warning"] + "⚠️ Some dependencies are missing!" + Style.RESET_ALL)
            self.loading_animation(1, "Preparing")
            
            install_cmd = [sys.executable, "-m", "pip", "install", "--upgrade"] + missing
            
            try:
                subprocess.check_call(install_cmd)
                print(COLORS["success"] + "✅ All dependencies installed successfully!" + Style.RESET_ALL)
                time.sleep(1)
            except subprocess.CalledProcessError:
                print(COLORS["error"] + "❌ Failed to install dependencies. Try manually: pip install " + " ".join(missing) + Style.RESET_ALL)
                time.sleep(2)
    
    def check_chrome_driver(self) -> bool:
        """Check Chrome Driver availability"""
        try:
            from selenium import webdriver
            driver = webdriver.Chrome()
            driver.quit()
            return True
        except WebDriverException as e:
            if "executable needs to be in PATH" in str(e):
                return False
            return True
        except:
            return False
    
    def install_chrome_driver(self) -> bool:
        """Automatically install Chrome Driver"""
        print(COLORS["info"] + "🔧 Attempting to install Chrome Driver..." + Style.RESET_ALL)
        
        try:
            import chromedriver_autoinstaller
            chromedriver_autoinstaller.install()
            return True
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to auto-install Chrome Driver: {str(e)}" + Style.RESET_ALL)
            print(COLORS["info"] + "ℹ️ Please download manually from https://chromedriver.chromium.org/" + Style.RESET_ALL)
            return False
    
    def check_update(self, force_notification: bool = True) -> None:
        """Check for updates from GitHub"""
        now = time.time()
        
        if not force_notification and (now - self.last_update_check) < UPDATE_INTERVAL:
            return
            
        self.last_update_check = now
        
        try:
            response = requests.get(REPO_URL, timeout=5)
            if response.status_code == 200:
                remote_version = None
                for line in response.text.split("\n"):
                    if line.startswith("# Version:"):
                        remote_version = line.split(":")[1].strip()
                        break
                
                if remote_version and remote_version != "2.1.0":
                    print(COLORS["info"] + f"🎉 Update available! Version {remote_version}" + Style.RESET_ALL)
                    if self.config["auto_update"] or self.ask_yes_no("Update now? (y/n): "):
                        self.update_script()
                elif force_notification:
                    print(COLORS["success"] + "✅ You're using the latest version!" + Style.RESET_ALL)
        except Exception as e:
            if force_notification:
                print(COLORS["error"] + f"❌ Failed to check for updates: {str(e)}" + Style.RESET_ALL)
    
    def update_script(self) -> None:
        """Automatically update the script"""
        print(COLORS["info"] + "🔄 Starting update process..." + Style.RESET_ALL)
        
        try:
            response = requests.get(REPO_URL, timeout=10)
            if response.status_code == 200:
                with open(__file__, "w", encoding="utf-8") as f:
                    f.write(response.text)
                
                print(COLORS["success"] + "✅ Update successful! Please restart the script." + Style.RESET_ALL)
                time.sleep(2)
                sys.exit(0)
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to update: {str(e)}" + Style.RESET_ALL)
    
    def ask_yes_no(self, question: str) -> bool:
        """Ask yes/no question with input validation"""
        while True:
            answer = input(COLORS["input"] + question + Style.RESET_ALL).lower()
            if answer in ("y", "yes"):
                return True
            elif answer in ("n", "no"):
                return False
            else:
                print(COLORS["error"] + "❌ Invalid input. Enter y/n" + Style.RESET_ALL)
    
    def get_input(self, prompt: str, password: bool = False) -> str:
        """Get user input with nice display"""
        color = random.choice(COLORS["neon"])
        prompt_text = f"{color}[?] {prompt}: {Style.RESET_ALL}"
        
        if password:
            return getpass(prompt_text)
        else:
            return input(prompt_text)
    
    def github_login(self) -> None:
        """Login to GitHub using Selenium"""
        print(COLORS["menu"] + "\n🔑 Login to GitHub\n" + Style.RESET_ALL)
        
        username = self.get_input("GitHub Username")
        password = self.get_input("Password", password=True)
        otp = None
        
        # Check if 2FA is enabled
        if self.ask_yes_no("Are you using 2FA? (y/n): "):
            otp = self.get_input("2FA OTP Code")
        
        print("\n" + COLORS["info"] + "🚀 Opening browser for login..." + Style.RESET_ALL)
        
        try:
            # Setup Chrome options
            chrome_options = Options()
            chrome_options.add_argument("--headless")
            chrome_options.add_argument("--no-sandbox")
            chrome_options.add_argument("--disable-dev-shm-usage")
            
            # Initialize driver
            self.driver = webdriver.Chrome(options=chrome_options)
            self.driver.get("https://github.com/login")
            
            # Fill login form
            username_field = WebDriverWait(self.driver, 10).until(
                EC.presence_of_element_located((By.ID, "login_field"))
            )
            username_field.send_keys(username)
            
            password_field = self.driver.find_element(By.ID, "password")
            password_field.send_keys(password)
            
            # Submit form
            self.driver.find_element(By.NAME, "commit").click()
            
            # Handle 2FA if enabled
            if otp:
                try:
                    otp_field = WebDriverWait(self.driver, 5).until(
                        EC.presence_of_element_located((By.ID, "otp"))
                    )
                    otp_field.send_keys(otp)
                    self.driver.find_element(By.XPATH, "//button[contains(text(),'Verify')]").click()
                except TimeoutException:
                    pass
            
            # Wait for successful login
            WebDriverWait(self.driver, 10).until(
                EC.presence_of_element_located((By.XPATH, "//span[contains(text(),'Pull requests')]"))
            )
            
            # Get cookies
            cookies = self.driver.get_cookies()
            for cookie in cookies:
                self.session.cookies.set(cookie['name'], cookie['value'])
            
            print(COLORS["success"] + "✅ Login successful!" + Style.RESET_ALL)
            
            # Save username to config
            self.config["github_username"] = username
            self.save_config()
            
        except Exception as e:
            print(COLORS["error"] + f"❌ Login failed: {str(e)}" + Style.RESET_ALL)
            if self.driver:
                self.driver.quit()
                self.driver = None
    
    def add_github_token(self) -> None:
        """Add GitHub Fine-grained token"""
        print(COLORS["menu"] + "\n🔑 Add GitHub Token\n" + Style.RESET_ALL)
        
        token_name = self.get_input("Token name (for identification)")
        token_value = self.get_input("Token value", password=True)
        token_permissions = self.get_input("Permissions (comma separated, e.g., repo,user)")
        
        # Validate token
        if not self.validate_github_token(token_value):
            print(COLORS["error"] + "❌ Invalid token or insufficient permissions!" + Style.RESET_ALL)
            return
        
        # Save token
        self.github_tokens["tokens"][token_name] = {
            "value": token_value,
            "permissions": [p.strip() for p in token_permissions.split(",")],
            "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        }
        
        # Set as active token if none is active
        if not self.github_tokens["active"]:
            self.github_tokens["active"] = token_name
        
        self.save_tokens()
        print(COLORS["success"] + f"✅ Token '{token_name}' added successfully!" + Style.RESET_ALL)
    
    def validate_github_token(self, token: str) -> bool:
        """Validate GitHub token"""
        headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        }
        
        try:
            response = requests.get("https://api.github.com/user", headers=headers)
            return response.status_code == 200
        except:
            return False
    
    def list_github_tokens(self) -> None:
        """List all saved GitHub tokens"""
        if not self.github_tokens["tokens"]:
            print(COLORS["warning"] + "⚠️ No saved tokens found!" + Style.RESET_ALL)
            return
        
        print(COLORS["menu"] + "\n🔑 List of GitHub Tokens\n" + Style.RESET_ALL)
        
        for name, data in self.github_tokens["tokens"].items():
            active = "(ACTIVE)" if self.github_tokens["active"] == name else ""
            color = COLORS["success"] if active else COLORS["info"]
            
            print(f"{color}• {name} {active}")
            print(f"  Created: {data['created_at']}")
            print(f"  Permissions: {', '.join(data['permissions'])}\n")
    
    def switch_github_token(self) -> None:
        """Switch active token"""
        if not self.github_tokens["tokens"]:
            print(COLORS["warning"] + "⚠️ No saved tokens found!" + Style.RESET_ALL)
            return
        
        self.list_github_tokens()
        
        token_name = self.get_input("Token name to activate")
        
        if token_name in self.github_tokens["tokens"]:
            self.github_tokens["active"] = token_name
            self.save_tokens()
            print(COLORS["success"] + f"✅ Token '{token_name}' activated successfully!" + Style.RESET_ALL)
        else:
            print(COLORS["error"] + "❌ Token not found!" + Style.RESET_ALL)
    
    def remove_github_token(self) -> None:
        """Remove saved token"""
        if not self.github_tokens["tokens"]:
            print(COLORS["warning"] + "⚠️ No saved tokens found!" + Style.RESET_ALL)
            return
        
        self.list_github_tokens()
        
        token_name = self.get_input("Token name to remove")
        
        if token_name in self.github_tokens["tokens"]:
            if self.github_tokens["active"] == token_name:
                self.github_tokens["active"] = None
            
            del self.github_tokens["tokens"][token_name]
            self.save_tokens()
            print(COLORS["success"] + f"✅ Token '{token_name}' removed successfully!" + Style.RESET_ALL)
        else:
            print(COLORS["error"] + "❌ Token not found!" + Style.RESET_ALL)
    
    def get_active_token(self) -> Optional[str]:
        """Get active token"""
        if not self.github_tokens["active"]:
            return None
        
        return self.github_tokens["tokens"][self.github_tokens["active"]]["value"]
    
    def create_github_repo(self) -> None:
        """Create new GitHub repository"""
        token = self.get_active_token()
        if not token:
            print(COLORS["error"] + "❌ No active token! Please add a token first." + Style.RESET_ALL)
            return
        
        print(COLORS["menu"] + "\n🆕 Create New Repository\n" + Style.RESET_ALL)
        
        repo_name = self.get_input("Repository name")
        description = self.get_input("Description (optional)", password=False) or ""
        is_private = self.ask_yes_no("Private repository? (y/n): ")
        
        data = {
            "name": repo_name,
            "description": description,
            "private": is_private,
            "auto_init": True
        }
        
        headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        }
        
        self.loading_animation(2, "Creating repository")
        
        try:
            response = requests.post(
                "https://api.github.com/user/repos",
                headers=headers,
                json=data
            )
            
            if response.status_code == 201:
                repo_url = response.json()["html_url"]
                self.config["last_repo"] = repo_url
                self.save_config()
                
                print(COLORS["success"] + f"✅ Repository created successfully! URL: {repo_url}" + Style.RESET_ALL)
                
                # Open in browser if desired
                if self.ask_yes_no("Open in browser? (y/n): "):
                    webbrowser.open(repo_url)
            else:
                error_msg = response.json().get("message", "Unknown error")
                print(COLORS["error"] + f"❌ Failed to create repository: {error_msg}" + Style.RESET_ALL)
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to create repository: {str(e)}" + Style.RESET_ALL)
    
    def list_github_repos(self) -> None:
        """List all user repositories"""
        token = self.get_active_token()
        if not token:
            print(COLORS["error"] + "❌ No active token! Please add a token first." + Style.RESET_ALL)
            return
        
        headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        }
        
        self.loading_animation(2, "Fetching repositories")
        
        try:
            response = requests.get(
                "https://api.github.com/user/repos",
                headers=headers,
                params={"per_page": 100}
            )
            
            if response.status_code == 200:
                repos = response.json()
                if not repos:
                    print(COLORS["warning"] + "⚠️ No repositories found!" + Style.RESET_ALL)
                    return
                
                print(COLORS["menu"] + "\n📂 List of Repositories\n" + Style.RESET_ALL)
                
                for repo in repos:
                    color = COLORS["neon"][0] if repo["private"] else COLORS["neon"][2]
                    print(f"{color}• {repo['name']} ({'Private' if repo['private'] else 'Public'})")
                    print(f"  {repo['description'] or 'No description'}")
                    print(f"  URL: {repo['html_url']}\n")
            else:
                error_msg = response.json().get("message", "Unknown error")
                print(COLORS["error"] + f"❌ Failed to fetch repositories: {error_msg}" + Style.RESET_ALL)
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to fetch repositories: {str(e)}" + Style.RESET_ALL)
    
    def upload_to_repository(self) -> None:
        """Upload document to GitHub repository"""
        token = self.get_active_token()
        if not token:
            print(COLORS["error"] + "❌ No active token! Please add a token first." + Style.RESET_ALL)
            return
        
        print(COLORS["menu"] + "\n📤 Upload Document to Repository\n" + Style.RESET_ALL)
        
        # Get list of repositories
        repos = self._get_user_repos(token)
        if not repos:
            return
        
        # Select repository
        print(COLORS["info"] + "📂 Your Repositories:" + Style.RESET_ALL)
        for i, repo in enumerate(repos, 1):
            print(f"{COLORS['neon'][i % len(COLORS['neon'])]}{i}. {repo['name']} ({'Private' if repo['private'] else 'Public'})")
        
        try:
            repo_choice = int(self.get_input("Select repository (number)")) - 1
            if repo_choice < 0 or repo_choice >= len(repos):
                print(COLORS["error"] + "❌ Invalid selection!" + Style.RESET_ALL)
                return
            
            selected_repo = repos[repo_choice]
        except ValueError:
            print(COLORS["error"] + "❌ Please enter a valid number!" + Style.RESET_ALL)
            return
        
        # Select file to upload
        file_path = self.get_input("Enter file path to upload")
        if not os.path.exists(file_path):
            print(COLORS["error"] + "❌ File not found!" + Style.RESET_ALL)
            return
        
        # Get target path in repository
        repo_path = self.get_input("Enter target path in repository (e.g., docs/file.txt)") or os.path.basename(file_path)
        
        # Read file content
        try:
            with open(file_path, "rb") as f:
                file_content = f.read()
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to read file: {str(e)}" + Style.RESET_ALL)
            return
        
        # Upload file using GitHub API
        url = f"https://api.github.com/repos/{selected_repo['full_name']}/contents/{repo_path}"
        headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        }
        
        data = {
            "message": f"Add {os.path.basename(file_path)} via GUTA",
            "content": base64.b64encode(file_content).decode("utf-8")
        }
        
        self.loading_animation(2, "Uploading file")
        
        try:
            response = requests.put(url, headers=headers, json=data)
            
            if response.status_code == 201:
                print(COLORS["success"] + f"✅ File uploaded successfully to {selected_repo['name']}!" + Style.RESET_ALL)
                print(f"URL: {response.json().get('content', {}).get('html_url', 'Not available')}")
            else:
                error_msg = response.json().get("message", "Unknown error")
                print(COLORS["error"] + f"❌ Failed to upload file: {error_msg}" + Style.RESET_ALL)
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to upload file: {str(e)}" + Style.RESET_ALL)

    def delete_repository(self) -> None:
        """Delete GitHub repository"""
        token = self.get_active_token()
        if not token:
            print(COLORS["error"] + "❌ No active token! Please add a token first." + Style.RESET_ALL)
            return
        
        print(COLORS["menu"] + "\n🗑️ Delete Repository\n" + Style.RESET_ALL)
        
        # Get list of repositories
        repos = self._get_user_repos(token)
        if not repos:
            return
        
        # Select repository
        print(COLORS["warning"] + "⚠️ Deleted repositories cannot be recovered!" + Style.RESET_ALL)
        print(COLORS["info"] + "📂 Your Repositories:" + Style.RESET_ALL)
        for i, repo in enumerate(repos, 1):
            print(f"{COLORS['neon'][i % len(COLORS['neon'])]}{i}. {repo['name']} ({'Private' if repo['private'] else 'Public'})")
        
        try:
            repo_choice = int(self.get_input("Select repository to delete (number)")) - 1
            if repo_choice < 0 or repo_choice >= len(repos):
                print(COLORS["error"] + "❌ Invalid selection!" + Style.RESET_ALL)
                return
            
            selected_repo = repos[repo_choice]
        except ValueError:
            print(COLORS["error"] + "❌ Please enter a valid number!" + Style.RESET_ALL)
            return
        
        # Confirm deletion
        confirm = self.get_input(f"Type '{selected_repo['name']}' to confirm deletion")
        if confirm != selected_repo['name']:
            print(COLORS["error"] + "❌ Confirmation mismatch. Deletion canceled." + Style.RESET_ALL)
            return
        
        # Delete repository using GitHub API
        url = f"https://api.github.com/repos/{selected_repo['full_name']}"
        headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        }
        
        self.loading_animation(2, "Deleting repository")
        
        try:
            response = requests.delete(url, headers=headers)
            
            if response.status_code == 204:
                print(COLORS["success"] + f"✅ Repository '{selected_repo['name']}' deleted successfully!" + Style.RESET_ALL)
            else:
                error_msg = response.json().get("message", "Unknown error")
                print(COLORS["error"] + f"❌ Failed to delete repository: {error_msg}" + Style.RESET_ALL)
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to delete repository: {str(e)}" + Style.RESET_ALL)

    def _get_user_repos(self, token: str) -> List[Dict]:
        """Get list of user repositories"""
        headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json"
        }
        
        try:
            response = requests.get(
                "https://api.github.com/user/repos",
                headers=headers,
                params={"per_page": 100}
            )
            
            if response.status_code == 200:
                return response.json()
            else:
                error_msg = response.json().get("message", "Unknown error")
                print(COLORS["error"] + f"❌ Failed to fetch repositories: {error_msg}" + Style.RESET_ALL)
                return []
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to fetch repositories: {str(e)}" + Style.RESET_ALL)
            return []

    def github_search(self) -> None:
        """Search on GitHub"""
        query = self.get_input("Enter search keywords")
        search_type = self.get_input("Search type (repositories/users/issues): ").lower()
        
        valid_types = ["repositories", "users", "issues"]
        if search_type not in valid_types:
            print(COLORS["error"] + "❌ Invalid search type!" + Style.RESET_ALL)
            return
        
        url = f"https://github.com/search?q={query}&type={search_type}"
        
        print(COLORS["info"] + f"🔍 Opening search results for '{query}'..." + Style.RESET_ALL)
        
        try:
            webbrowser.open(url)
        except Exception as e:
            print(COLORS["error"] + f"❌ Failed to open browser: {str(e)}" + Style.RESET_ALL)
    
    def clone_repo(self) -> None:
        """Clone GitHub repository"""
        repo_url = self.get_input("Enter repository URL (e.g., https://github.com/user/repo)")
        
        if not repo_url.startswith("https://github.com/"):
            print(COLORS["error"] + "❌ Invalid GitHub repository URL!" + Style.RESET_ALL)
            return
        
        repo_name = repo_url.split("/")[-1]
        if repo_name.endswith(".git"):
            repo_name = repo_name[:-4]
        
        clone_dir = os.path.join(os.getcwd(), repo_name)
        
        if os.path.exists(clone_dir):
            print(COLORS["error"] + f"❌ Directory '{repo_name}' already exists!" + Style.RESET_ALL)
            return
        
        print(COLORS["info"] + f"⏬ Cloning repository to '{clone_dir}'..." + Style.RESET_ALL)
        
        try:
            subprocess.run(["git", "clone", repo_url], check=True)
            print(COLORS["success"] + "✅ Repository cloned successfully!" + Style.RESET_ALL)
        except subprocess.CalledProcessError as e:
            print(COLORS["error"] + f"❌ Failed to clone repository: {str(e)}" + Style.RESET_ALL)
    
    def show_settings(self) -> None:
        """Display and modify settings"""
        print(COLORS["menu"] + "\n⚙️ Settings\n" + Style.RESET_ALL)
        
        for key, value in self.config.items():
            color = COLORS["neon"][1] if isinstance(value, bool) else COLORS["neon"][3]
            print(f"{color}• {key}: {value}")
        
        print()
        
        setting = self.get_input("Enter setting name to modify (leave blank to cancel)").strip()
        
        if not setting:
            return
        
        if setting not in self.config:
            print(COLORS["error"] + "❌ Setting not found!" + Style.RESET_ALL)
            return
        
        current_value = self.config[setting]
        
        if isinstance(current_value, bool):
            new_value = self.ask_yes_no(f"Change {setting} to {'false' if current_value else 'true'}? (y/n): ")
            self.config[setting] = new_value
        elif isinstance(current_value, int):
            try:
                new_value = int(self.get_input(f"Enter new value for {setting} (integer)"))
                self.config[setting] = new_value
            except ValueError:
                print(COLORS["error"] + "❌ Value must be a number!" + Style.RESET_ALL)
                return
        else:
            new_value = self.get_input(f"Enter new value for {setting}")
            self.config[setting] = new_value
        
        self.save_config()
        print(COLORS["success"] + f"✅ Setting '{setting}' updated successfully!" + Style.RESET_ALL)
    
    def show_menu(self) -> None:
        """Display main menu"""
        menu_items = [
            {"option": "1", "text": "Login to GitHub (Browser)", "action": self.github_login},
            {"option": "2", "text": "Add GitHub Token", "action": self.add_github_token},
            {"option": "3", "text": "List GitHub Tokens", "action": self.list_github_tokens},
            {"option": "4", "text": "Switch Active Token", "action": self.switch_github_token},
            {"option": "5", "text": "Remove GitHub Token", "action": self.remove_github_token},
            {"option": "6", "text": "Create Repository", "action": self.create_github_repo},
            {"option": "7", "text": "List Repositories", "action": self.list_github_repos},
            {"option": "8", "text": "Upload to Repository", "action": self.upload_to_repository},
            {"option": "9", "text": "Delete Repository", "action": self.delete_repository},
            {"option": "10", "text": "Search on GitHub", "action": self.github_search},
            {"option": "11", "text": "Clone Repository", "action": self.clone_repo},
            {"option": "12", "text": "Settings", "action": self.show_settings},
            {"option": "13", "text": "Check for Updates", "action": lambda: self.check_update(force_notification=True)},
            {"option": "0", "text": "Exit", "action": exit}
        ]
        
        while True:
            print(COLORS["menu"] + "\n✨ GUTA Main Menu\n" + Style.RESET_ALL)
            
            for item in menu_items:
                color = random.choice(COLORS["neon"])
                print(f"{color}[{item['option']}] {item['text']}")
            
            print()
            
            choice = self.get_input("Select menu").strip()
            
            for item in menu_items:
                if choice == item["option"]:
                    try:
                        item["action"]()
                    except Exception as e:
                        print(COLORS["error"] + f"❌ Error occurred: {str(e)}" + Style.RESET_ALL)
                        self.log_error(str(e))
                    break
            else:
                print(COLORS["error"] + "❌ Invalid selection!" + Style.RESET_ALL)
            
            input(COLORS["input"] + "\nPress Enter to continue..." + Style.RESET_ALL)
            self.clear_screen()
            self.show_banner()
    
    def run(self) -> None:
        """Run main application"""
        try:
            self.check_dependencies()
            
            if not self.check_chrome_driver():
                if self.ask_yes_no("Chrome Driver not found. Install now? (y/n): "):
                    self.install_chrome_driver()
            
            if self.config["show_banner"]:
                self.show_banner()
            else:
                self.clear_screen()
            
            self.show_menu()
        except KeyboardInterrupt:
            print(COLORS["error"] + "\n❌ Application stopped by user" + Style.RESET_ALL)
            if self.driver:
                self.driver.quit()
            sys.exit(0)
        except Exception as e:
            print(COLORS["error"] + f"❌ Fatal error: {str(e)}" + Style.RESET_ALL)
            self.log_error(f"Fatal error: {str(e)}")
            if self.driver:
                self.driver.quit()
            sys.exit(1)

if __name__ == "__main__":
    app = GUTA()
    app.run()