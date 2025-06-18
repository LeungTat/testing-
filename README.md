```
#!/usr/bin/env python3
"""
Markdown to PDF Converter with Image Support
Supports both Windows and Linux environments
Uses only pip-installable dependencies
"""

import os
import sys
import re
import base64
import urllib.request
from pathlib import Path
from typing import List, Tuple, Optional
import argparse

try:
    import markdown
    from markdown.extensions import codehilite, tables, toc
except ImportError:
    print("Installing required packages...")
    os.system(f"{sys.executable} -m pip install markdown")
    import markdown
    from markdown.extensions import codehilite, tables, toc

try:
    from weasyprint import HTML, CSS
    from weasyprint.css import get_all_computed_styles
    from weasyprint.css.targets import TargetCollector
except ImportError:
    print("Installing weasyprint...")
    os.system(f"{sys.executable} -m pip install weasyprint")
    from weasyprint import HTML, CSS

try:
    from PIL import Image
except ImportError:
    print("Installing Pillow...")
    os.system(f"{sys.executable} -m pip install Pillow")
    from PIL import Image

import tempfile
import shutil


class MarkdownToPDFConverter:
    """
    Converts Markdown files with images to PDF
    Handles both local and remote images
    """
    
    def __init__(self, css_style: Optional[str] = None):
        """
        Initialize the converter
        
        Args:
            css_style: Optional custom CSS styling
        """
        self.css_style = css_style or self._get_default_css()
        self.temp_dir = None
        
    def _get_default_css(self) -> str:
        """Get default CSS styling for the PDF"""
        return """
        @page {
            size: A4;
            margin: 2cm;
        }
        
        body {
            font-family: 'DejaVu Sans', Arial, sans-serif;
            font-size: 12pt;
            line-height: 1.6;
            color: #333;
            max-width: 100%;
        }
        
        h1, h2, h3, h4, h5, h6 {
            color: #2c3e50;
            margin-top: 1.5em;
            margin-bottom: 0.5em;
        }
        
        h1 { font-size: 24pt; border-bottom: 2px solid #3498db; padding-bottom: 0.3em; }
        h2 { font-size: 20pt; border-bottom: 1px solid #bdc3c7; padding-bottom: 0.2em; }
        h3 { font-size: 16pt; }
        h4 { font-size: 14pt; }
        
        p {
            margin-bottom: 1em;
            text-align: justify;
        }
        
        img {
            max-width: 100%;
            height: auto;
            display: block;
            margin: 1em auto;
            border: 1px solid #ddd;
            border-radius: 4px;
            padding: 5px;
        }
        
        code {
            background-color: #f8f9fa;
            padding: 0.2em 0.4em;
            border-radius: 3px;
            font-family: 'DejaVu Sans Mono', monospace;
            font-size: 0.9em;
        }
        
        pre {
            background-color: #f8f9fa;
            border: 1px solid #e9ecef;
            border-radius: 4px;
            padding: 1em;
            overflow-x: auto;
            font-family: 'DejaVu Sans Mono', monospace;
            font-size: 0.85em;
        }
        
        pre code {
            background-color: transparent;
            padding: 0;
            border-radius: 0;
        }
        
        blockquote {
            border-left: 4px solid #3498db;
            padding-left: 1em;
            margin: 1em 0;
            font-style: italic;
            color: #555;
        }
        
        table {
            border-collapse: collapse;
            width: 100%;
            margin: 1em 0;
        }
        
        th, td {
            border: 1px solid #ddd;
            padding: 0.5em;
            text-align: left;
        }
        
        th {
            background-color: #f8f9fa;
            font-weight: bold;
        }
        
        ul, ol {
            margin: 1em 0;
            padding-left: 2em;
        }
        
        li {
            margin-bottom: 0.5em;
        }
        
        a {
            color: #3498db;
            text-decoration: none;
        }
        
        a:hover {
            text-decoration: underline;
        }
        """
    
    def _setup_temp_directory(self):
        """Create temporary directory for processing"""
        if self.temp_dir is None:
            self.temp_dir = tempfile.mkdtemp()
    
    def _cleanup_temp_directory(self):
        """Clean up temporary directory"""
        if self.temp_dir and os.path.exists(self.temp_dir):
            shutil.rmtree(self.temp_dir)
            self.temp_dir = None
    
    def _process_images(self, markdown_content: str, base_path: str) -> str:
        """
        Process images in markdown content
        Convert relative paths to absolute paths and handle remote images
        
        Args:
            markdown_content: The markdown content
            base_path: Base path for resolving relative image paths
            
        Returns:
            Modified markdown content with processed image paths
        """
        self._setup_temp_directory()
        
        # Pattern to match markdown images: ![alt text](image_path "optional title")
        img_pattern = r'!\[([^\]]*)\]\(([^)]+)(?:\s+"([^"]*)")?\)'
        
        def replace_image(match):
            alt_text = match.group(1)
            img_path = match.group(2)
            title = match.group(3) or ""
            
            try:
                # Handle remote images
                if img_path.startswith(('http://', 'https://')):
                    return self._download_remote_image(img_path, alt_text, title)
                
                # Handle local images
                else:
                    # Convert relative path to absolute
                    if not os.path.isabs(img_path):
                        img_path = os.path.join(base_path, img_path)
                    
                    if os.path.exists(img_path):
                        # Copy to temp directory to ensure access
                        filename = os.path.basename(img_path)
                        temp_path = os.path.join(self.temp_dir, filename)
                        shutil.copy2(img_path, temp_path)
                        
                        # Return with updated path
                        title_attr = f' "{title}"' if title else ""
                        return f'![{alt_text}](file://{os.path.abspath(temp_path)}{title_attr})'
                    else:
                        print(f"Warning: Image not found: {img_path}")
                        return match.group(0)  # Return original if not found
                        
            except Exception as e:
                print(f"Error processing image {img_path}: {e}")
                return match.group(0)  # Return original on error
        
        return re.sub(img_pattern, replace_image, markdown_content)
    
    def _download_remote_image(self, url: str, alt_text: str, title: str) -> str:
        """
        Download remote image and return markdown with local path
        
        Args:
            url: Remote image URL
            alt_text: Alt text for the image
            title: Title for the image
            
        Returns:
            Markdown image syntax with local path
        """
        try:
            # Generate filename from URL
            filename = os.path.basename(url.split('?')[0])  # Remove query params
            if not filename or '.' not in filename:
                filename = f"image_{hash(url) % 10000}.jpg"
            
            temp_path = os.path.join(self.temp_dir, filename)
            
            # Download image
            urllib.request.urlretrieve(url, temp_path)
            
            # Verify it's a valid image
            with Image.open(temp_path) as img:
                img.verify()
            
            title_attr = f' "{title}"' if title else ""
            return f'![{alt_text}](file://{os.path.abspath(temp_path)}{title_attr})'
            
        except Exception as e:
            print(f"Error downloading image {url}: {e}")
            return f'![{alt_text}]({url}){"" if not title else f" ({title})"}'
    
    def convert_markdown_to_html(self, markdown_content: str) -> str:
        """
        Convert markdown content to HTML
        
        Args:
            markdown_content: The markdown content to convert
            
        Returns:
            HTML content
        """
        # Configure markdown extensions
        extensions = [
            'markdown.extensions.extra',  # Includes tables, footnotes, etc.
            'markdown.extensions.codehilite',  # Syntax highlighting
            'markdown.extensions.toc',  # Table of contents
            'markdown.extensions.nl2br',  # Newline to break
        ]
        
        # Create markdown instance
        md = markdown.Markdown(extensions=extensions)
        
        # Convert to HTML
        html_content = md.convert(markdown_content)
        
        # Wrap in full HTML document
        full_html = f"""
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8">
            <title>Converted Document</title>
        </head>
        <body>
            {html_content}
        </body>
        </html>
        """
        
        return full_html
    
    def convert_file(self, input_file: str, output_file: str) -> bool:
        """
        Convert a markdown file to PDF
        
        Args:
            input_file: Path to input markdown file
            output_file: Path to output PDF file
            
        Returns:
            True if successful, False otherwise
        """
        try:
            # Read markdown file
            with open(input_file, 'r', encoding='utf-8') as f:
                markdown_content = f.read()
            
            return self.convert_content(markdown_content, output_file, os.path.dirname(os.path.abspath(input_file)))
            
        except Exception as e:
            print(f"Error converting file {input_file}: {e}")
            return False
        finally:
            self._cleanup_temp_directory()
    
    def convert_content(self, markdown_content: str, output_file: str, base_path: str = ".") -> bool:
        """
        Convert markdown content to PDF
        
        Args:
            markdown_content: The markdown content to convert
            output_file: Path to output PDF file
            base_path: Base path for resolving relative image paths
            
        Returns:
            True if successful, False otherwise
        """
        try:
            # Process images in markdown
            processed_content = self._process_images(markdown_content, base_path)
            
            # Convert to HTML
            html_content = self.convert_markdown_to_html(processed_content)
            
            # Create PDF
            html_doc = HTML(string=html_content)
            css_doc = CSS(string=self.css_style)
            
            html_doc.write_pdf(output_file, stylesheets=[css_doc])
            
            print(f"Successfully converted to PDF: {output_file}")
            return True
            
        except Exception as e:
            print(f"Error converting content: {e}")
            return False
        finally:
            self._cleanup_temp_directory()


def main():
    """Main function to handle command line arguments"""
    parser = argparse.ArgumentParser(description='Convert Markdown files with images to PDF')
    parser.add_argument('input', help='Input markdown file path')
    parser.add_argument('-o', '--output', help='Output PDF file path (default: same name as input with .pdf extension)')
    parser.add_argument('--css', help='Custom CSS file path')
    
    args = parser.parse_args()
    
    # Determine output file path
    if args.output:
        output_file = args.output
    else:
        base_name = os.path.splitext(args.input)[0]
        output_file = f"{base_name}.pdf"
    
    # Load custom CSS if provided
    css_style = None
    if args.css and os.path.exists(args.css):
        with open(args.css, 'r', encoding='utf-8') as f:
            css_style = f.read()
    
    # Create converter and convert file
    converter = MarkdownToPDFConverter(css_style)
    success = converter.convert_file(args.input, output_file)
    
    if success:
        print(f"Conversion completed successfully!")
        print(f"Output: {os.path.abspath(output_file)}")
    else:
        print("Conversion failed!")
        sys.exit(1)


if __name__ == "__main__":
    main()
```


# Install dependencies
pip install markdown reportlab Pillow html2text

# Test with sample file
python md_to_pdf_reportlab.py sample.md

# Or use WeasyPrint version (if system deps available)
pip install weasyprint
python new.py sample.md
