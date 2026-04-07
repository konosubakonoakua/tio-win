# tio-win
tio on windows based on msys2, this repo is only for releases.
<img width="1029" height="146" alt="image" src="https://github.com/user-attachments/assets/267bcced-c808-428d-87a3-17c758b7e02d" />

<details>

```python
#!/usr/bin/env python3
# extract_tio_dynamic.py

import re
import requests
import os
import sys
import tempfile
import tarfile
import zstandard as zstd

def get_latest_tio_url():
    """Scrape the packages.msys2.org page for the latest tio x86_64 download link."""
    url = "https://packages.msys2.org/packages/tio?variant=x86_64"
    
    try:
        response = requests.get(url)
        response.raise_for_status()
        
        # Look for the .pkg.tar.zst link in the page
        pattern = r'href="(https://mirror\.msys2\.org/msys/x86_64/tio-\d+\.\d+-\d+-x86_64\.pkg\.tar\.zst)"'
        match = re.search(pattern, response.text)
        
        if not match:
            raise ValueError("Download link not found on packages page")
        
        download_url = match.group(1)
        filename = os.path.basename(download_url)
        
        print(f"[*] Found latest package: {filename}")
        return download_url, filename
        
    except Exception as e:
        print(f"[!] Failed to scrape packages page: {e}")
        sys.exit(1)

def download_file(url, local_path):
    """Download file from URL to local path."""
    print(f"[*] Downloading: {url}")
    try:
        response = requests.get(url, stream=True, timeout=60)
        response.raise_for_status()
        
        total_size = int(response.headers.get('content-length', 0))
        with open(local_path, 'wb') as f:
            downloaded = 0
            for chunk in response.iter_content(chunk_size=8192):
                if chunk:
                    f.write(chunk)
                    downloaded += len(chunk)
                    if total_size > 0:
                        percent = (downloaded / total_size) * 100
                        sys.stdout.write(f"\r  Progress: {percent:.1f}%")
                        sys.stdout.flush()
        
        print(f"\n[✓] Download complete: {local_path}")
        return True
        
    except Exception as e:
        print(f"\n[!] Download failed: {e}")
        return False

def extract_zst_to_tar(zst_path, tar_path):
    """Extract .zst file to .tar file."""
    print(f"[*] Decompressing zstd archive...")
    try:
        with open(zst_path, 'rb') as f_in, open(tar_path, 'wb') as f_out:
            dctx = zstd.ZstdDecompressor()
            dctx.copy_stream(f_in, f_out)
        print(f"[✓] Decompressed to: {tar_path}")
        return True
    except Exception as e:
        print(f"[!] Zstd decompression failed: {e}")
        return False

def extract_tar_and_find_exe(tar_path, output_dir):
    """Extract tar archive and find tio.exe."""
    print(f"[*] Extracting tar archive...")
    try:
        with tarfile.open(tar_path, 'r') as tar:
            # Find tio.exe in archive
            target_files = [m for m in tar.getmembers() if m.name.endswith('usr/bin/tio.exe')]
            
            if not target_files:
                print(f"[!] tio.exe not found in archive")
                return None
            
            # Extract tio.exe
            member = target_files[0]
            print(f"  [-] Extracting: {member.name}")
            tar.extract(member, path=output_dir)
            
            # Get full path of extracted file
            extracted_path = os.path.join(output_dir, member.name)
            final_path = os.path.join(output_dir, "tio.exe")
            
            # Rename to final path
            os.rename(extracted_path, final_path)
            print(f"[✓] Extracted to: {final_path}")
            return final_path
            
    except Exception as e:
        print(f"[!] Tar extraction failed: {e}")
        return None

def main():
    # Get latest download URL dynamically
    download_url, filename = get_latest_tio_url()
    
    # Configuration
    OUTPUT_DIR = r"C:\tio"
    
    # Create output directory
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    
    # Create temp directory
    with tempfile.TemporaryDirectory() as temp_dir:
        zst_path = os.path.join(temp_dir, filename)
        tar_path = os.path.join(temp_dir, filename.replace('.zst', ''))
        
        # Step 1: Download
        if not download_file(download_url, zst_path):
            sys.exit(1)
        
        # Step 2: Decompress zstd
        if not extract_zst_to_tar(zst_path, tar_path):
            sys.exit(1)
        
        # Step 3: Extract tar and find tio.exe
        exe_path = extract_tar_and_find_exe(tar_path, OUTPUT_DIR)
        
        if exe_path and os.path.exists(exe_path):
            print(f"\n[✓] SUCCESS! tio.exe extracted to: {exe_path}")
        else:
            print(f"\n[!] Failed to extract tio.exe")
            sys.exit(1)

if __name__ == "__main__":
    # Install required packages if not present
    try:
        import requests
        import zstandard
    except ImportError:
        print("[*] Installing required packages...")
        import subprocess
        subprocess.check_call([sys.executable, "-m", "pip", "install", "requests", "zstandard"])
    
    main()
```

</details>
