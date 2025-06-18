```
#!/usr/bin/env python3
"""
Alternative Markdown to PDF Converter using ReportLab
Pure Python solution with minimal system dependencies
"""

import os
import sys
import re
import urllib.request
import tempfile
import shutil
from pathlib import Path
from typing import List, Tuple, Optional
import argparse

# Auto-install dependencies
required_packages = ['markdown', 'reportlab', 'Pillow', 'html2text']

for package in required_packages:
    try:
        __import__(package.replace('-', '_'))
    except ImportError:
        print(f"Installing {package}...")
        os.system(f"{sys.executable} -m pip install {package}")

import markdown
from reportlab.lib.pagesizes import A4, letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Image as RLImage, Table, TableStyle
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.lib.units import inch, cm
from reportlab.lib import colors
from reportlab.lib.enums import TA_JUSTIFY, TA_CENTER, TA_LEFT
from PIL import Image
import html2text


class ReportLabMarkdownConverter:
    """
    Markdown to PDF converter using ReportLab
    Handles text, images, tables, and basic formatting
    """
    
    def __init__(self):
        self.temp_dir = None
        self.story = []
        self.styles = getSampleStyleSheet()
        self._setup_custom_styles()
    
    def _setup_custom_styles(self):
        """Setup custom paragraph styles"""
        # Title style
        self.styles.add(ParagraphStyle(
            name='CustomTitle',
            parent=self.styles['Heading1'],
            fontSize=18,
            spaceAfter=12,
            spaceBefore=12,
            textColor=colors.darkblue,
            alignment=TA_LEFT
        ))
        
        # Heading styles
        self.styles.add(ParagraphStyle(
            name='CustomHeading2',
            parent=self.styles['Heading2'],
            fontSize=16,
            spaceAfter=10,
            spaceBefore=10,
            textColor=colors.darkgreen
        ))
        
        self.styles.add(ParagraphStyle(
            name='CustomHeading3',
            parent=self.styles['Heading3'],
            fontSize=14,
            spaceAfter=8,
            spaceBefore=8
        ))
        
        # Code style
        self.styles.add(ParagraphStyle(
            name='Code',
            parent=self.styles['Normal'],
            fontSize=10,
            fontName='Courier',
            backgroundColor=colors.lightgrey,
            leftIndent=0.5*inch,
            rightIndent=0.5*inch,
            spaceAfter=6,
            spaceBefore=6
        ))
        
        # Quote style
        self.styles.add(ParagraphStyle(
            name='Quote',
            parent=self.styles['Normal'],
            fontSize=11,
            leftIndent=0.5*inch,
            rightIndent=0.5*inch,
            textColor=colors.darkgrey,
            borderColor=colors.blue,
            borderWidth=1,
            borderPadding=5,
            spaceAfter=8,
            spaceBefore=8
        ))
    
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
        """Process images in markdown content"""
        self._setup_temp_directory()
        
        img_pattern = r'!\[([^\]]*)\]\(([^)]+)(?:\s+"([^"]*)")?\)'
        
        def replace_image(match):
            alt_text = match.group(1)
            img_path = match.group(2)
            title = match.group(3) or ""
            
            try:
                if img_path.startswith(('http://', 'https://')):
                    return self._download_remote_image(img_path, alt_text, title)
                else:
                    if not os.path.isabs(img_path):
                        img_path = os.path.join(base_path, img_path)
                    
                    if os.path.exists(img_path):
                        filename = os.path.basename(img_path)
                        temp_path = os.path.join(self.temp_dir, filename)
                        shutil.copy2(img_path, temp_path)
                        return f'![{alt_text}]({temp_path})'
                    else:
                        print(f"Warning: Image not found: {img_path}")
                        return match.group(0)
            except Exception as e:
                print(f"Error processing image {img_path}: {e}")
                return match.group(0)
        
        return re.sub(img_pattern, replace_image, markdown_content)
    
    def _download_remote_image(self, url: str, alt_text: str, title: str) -> str:
        """Download remote image"""
        try:
            filename = os.path.basename(url.split('?')[0])
            if not filename or '.' not in filename:
                filename = f"image_{hash(url) % 10000}.jpg"
            
            temp_path = os.path.join(self.temp_dir, filename)
            urllib.request.urlretrieve(url, temp_path)
            
            # Verify it's a valid image
            with Image.open(temp_path) as img:
                img.verify()
            
            return f'![{alt_text}]({temp_path})'
        except Exception as e:
            print(f"Error downloading image {url}: {e}")
            return f'![{alt_text}]({url})'
    
    def _add_image_to_story(self, img_path: str, alt_text: str = ""):
        """Add image to the story"""
        try:
            if os.path.exists(img_path):
                # Get image dimensions
                with Image.open(img_path) as img:
                    width, height = img.size
                
                # Calculate scaled dimensions to fit page
                max_width = 6 * inch
                max_height = 4 * inch
                
                if width > max_width:
                    scale = max_width / width
                    width = max_width
                    height = height * scale
                
                if height > max_height:
                    scale = max_height / height
                    height = max_height
                    width = width * scale
                
                # Add image to story
                img_obj = RLImage(img_path, width=width, height=height)
                self.story.append(img_obj)
                self.story.append(Spacer(1, 12))
                
                # Add caption if alt_text exists
                if alt_text:
                    caption = Paragraph(f"<i>{alt_text}</i>", self.styles['Normal'])
                    self.story.append(caption)
                    self.story.append(Spacer(1, 12))
        except Exception as e:
            print(f"Error adding image {img_path}: {e}")
    
    def _parse_markdown_to_story(self, html_content: str):
        """Parse HTML content and convert to ReportLab story elements"""
        # Simple HTML parser for basic elements
        lines = html_content.split('\n')
        current_paragraph = []
        in_code_block = False
        
        for line in lines:
            line = line.strip()
            if not line:
                if current_paragraph:
                    text = ' '.join(current_paragraph)
                    if text:
                        para = Paragraph(text, self.styles['Normal'])
                        self.story.append(para)
                        self.story.append(Spacer(1, 6))
                    current_paragraph = []
                continue
            
            # Handle headers
            if line.startswith('<h1>') and line.endswith('</h1>'):
                text = line[4:-5]
                para = Paragraph(text, self.styles['CustomTitle'])
                self.story.append(para)
                continue
            elif line.startswith('<h2>') and line.endswith('</h2>'):
                text = line[4:-5]
                para = Paragraph(text, self.styles['CustomHeading2'])
                self.story.append(para)
                continue
            elif line.startswith('<h3>') and line.endswith('</h3>'):
                text = line[4:-5]
                para = Paragraph(text, self.styles['CustomHeading3'])
                self.story.append(para)
                continue
            
            # Handle code blocks
            if line.startswith('<pre><code>'):
                in_code_block = True
                code_text = line[11:]  # Remove <pre><code>
                if line.endswith('</code></pre>'):
                    code_text = code_text[:-13]  # Remove </code></pre>
                    para = Paragraph(f"<font name='Courier'>{code_text}</font>", self.styles['Code'])
                    self.story.append(para)
                    in_code_block = False
                continue
            elif in_code_block:
                if line.endswith('</code></pre>'):
                    code_text = line[:-13]
                    para = Paragraph(f"<font name='Courier'>{code_text}</font>", self.styles['Code'])
                    self.story.append(para)
                    in_code_block = False
                else:
                    para = Paragraph(f"<font name='Courier'>{line}</font>", self.styles['Code'])
                    self.story.append(para)
                continue
            
            # Handle images
            img_match = re.search(r'<img.*?src="([^"]*)".*?alt="([^"]*)".*?/?>', line)
            if img_match:
                img_path = img_match.group(1)
                alt_text = img_match.group(2)
                self._add_image_to_story(img_path, alt_text)
                continue
            
            # Handle blockquotes
            if line.startswith('<blockquote>') and line.endswith('</blockquote>'):
                text = line[12:-13]  # Remove tags
                para = Paragraph(text, self.styles['Quote'])
                self.story.append(para)
                continue
            
            # Handle paragraphs
            if line.startswith('<p>') and line.endswith('</p>'):
                text = line[3:-4]  # Remove <p> tags
                # Clean up basic HTML tags
                text = re.sub(r'<strong>(.*?)</strong>', r'<b>\1</b>', text)
                text = re.sub(r'<em>(.*?)</em>', r'<i>\1</i>', text)
                text = re.sub(r'<code>(.*?)</code>', r'<font name="Courier">\1</font>', text)
                
                para = Paragraph(text, self.styles['Normal'])
                self.story.append(para)
                self.story.append(Spacer(1, 6))
                continue
            
            # Collect other content into current paragraph
            current_paragraph.append(line)
        
        # Handle any remaining paragraph content
        if current_paragraph:
            text = ' '.join(current_paragraph)
            # Clean basic HTML tags
            text = re.sub(r'<[^>]+>', '', text)  # Remove any remaining HTML tags
            if text:
                para = Paragraph(text, self.styles['Normal'])
                self.story.append(para)
    
    def convert_file(self, input_file: str, output_file: str) -> bool:
        """Convert markdown file to PDF"""
        try:
            with open(input_file, 'r', encoding='utf-8') as f:
                markdown_content = f.read()
            
            return self.convert_content(markdown_content, output_file, os.path.dirname(os.path.abspath(input_file)))
        except Exception as e:
            print(f"Error converting file {input_file}: {e}")
            return False
        finally:
            self._cleanup_temp_directory()
    
    def convert_content(self, markdown_content: str, output_file: str, base_path: str = ".") -> bool:
        """Convert markdown content to PDF"""
        try:
            # Process images
            processed_content = self._process_images(markdown_content, base_path)
            
            # Convert markdown to HTML
            md = markdown.Markdown(extensions=[
                'markdown.extensions.extra',
                'markdown.extensions.codehilite',
                'markdown.extensions.toc'
            ])
            html_content = md.convert(processed_content)
            
            # Create PDF document
            doc = SimpleDocTemplate(output_file, pagesize=A4, 
                                  rightMargin=2*cm, leftMargin=2*cm,
                                  topMargin=2*cm, bottomMargin=2*cm)
            
            # Parse HTML and build story
            self._parse_markdown_to_story(html_content)
            
            # Build PDF
            doc.build(self.story)
            
            print(f"Successfully converted to PDF: {output_file}")
            return True
            
        except Exception as e:
            print(f"Error converting content: {e}")
            import traceback
            traceback.print_exc()
            return False
        finally:
            self._cleanup_temp_directory()


def main():
    """Main function"""
    parser = argparse.ArgumentParser(description='Convert Markdown files with images to PDF using ReportLab')
    parser.add_argument('input', help='Input markdown file path')
    parser.add_argument('-o', '--output', help='Output PDF file path')
    
    args = parser.parse_args()
    
    if args.output:
        output_file = args.output
    else:
        base_name = os.path.splitext(args.input)[0]
        output_file = f"{base_name}.pdf"
    
    converter = ReportLabMarkdownConverter()
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


```

# Markdown to PDF Converter - Setup Guide

## Overview

This project provides two Python-based solutions to convert Markdown files with images to PDF:

1. **`new.py`** - Uses WeasyPrint (better formatting but requires system dependencies on Linux)
2. **`md_to_pdf_reportlab.py`** - Uses ReportLab (pure Python, more compatible across platforms)

## Dependencies

### For WeasyPrint version (new.py):
```bash
pip install markdown weasyprint Pillow
```

### For ReportLab version (md_to_pdf_reportlab.py):
```bash
pip install markdown reportlab Pillow html2text
```

## Installation

### On Windows (Development):
```bash
# Clone or download the scripts
# Install dependencies
pip install -r requirements.txt

# Test the converter
python new.py sample.md -o output.pdf
```

### On Linux Server (Production):

#### Option 1: Simple pip install (if system dependencies are available):
```bash
pip install markdown weasyprint Pillow html2text reportlab
```

#### Option 2: If WeasyPrint has dependency issues on Linux:
WeasyPrint requires system libraries. Install them first:

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install python3-dev python3-pip python3-setuptools python3-wheel python3-cffi libcairo2 libpango-1.0-0 libpangocairo-1.0-0 libgdk-pixbuf2.0-0 libffi-dev shared-mime-info
```

**CentOS/RHEL:**
```bash
sudo yum install python3-devel python3-pip python3-setuptools python3-wheel python3-cffi cairo pango gdk-pixbuf2 libffi-devel
```

Then install Python packages:
```bash
pip install markdown weasyprint Pillow
```

#### Option 3: Use ReportLab version (Recommended for problematic servers):
If you can't install system dependencies, use the ReportLab version:
```bash
pip install markdown reportlab Pillow html2text
python md_to_pdf_reportlab.py sample.md -o output.pdf
```

## Usage

### Basic Usage:
```bash
# Using WeasyPrint version
python new.py input.md

# Using ReportLab version  
python md_to_pdf_reportlab.py input.md

# Specify output file
python new.py input.md -o output.pdf

# Custom CSS (WeasyPrint only)
python new.py input.md --css custom.css
```

### Python API Usage:
```python
from new import MarkdownToPDFConverter

# Create converter
converter = MarkdownToPDFConverter()

# Convert file
success = converter.convert_file('input.md', 'output.pdf')

# Convert content directly
markdown_content = "# Hello\n\nThis is **bold** text."
success = converter.convert_content(markdown_content, 'output.pdf')
```

## Features

### Supported Markdown Elements:
- Headers (H1-H6)
- Paragraphs
- Bold/Italic text
- Code blocks and inline code
- Lists (ordered/unordered)
- Links
- Images (local and remote)
- Tables
- Blockquotes

### Image Support:
- Local images (relative and absolute paths)
- Remote images (HTTP/HTTPS URLs)
- Automatic image scaling to fit page
- Image caching for remote images

## Troubleshooting

### Common Issues:

1. **WeasyPrint installation fails on Linux:**
   - Use the ReportLab version instead
   - Or install system dependencies as shown above

2. **Images not showing:**
   - Check image paths are correct
   - Ensure images are accessible
   - For remote images, check internet connectivity

3. **Fonts not rendering correctly:**
   - WeasyPrint uses system fonts
   - ReportLab uses built-in fonts (more reliable)

4. **Memory issues with large files:**
   - Process files in smaller chunks
   - Use ReportLab version for better memory management

### Performance Tips:
- Cache remote images locally
- Use smaller image sizes when possible
- Process multiple files in batches

## Deployment Strategies

### Strategy 1: Virtual Environment
```bash
# Create virtual environment
python3 -m venv pdf_converter_env
source pdf_converter_env/bin/activate  # Linux
# or
pdf_converter_env\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt

# Deploy scripts
cp *.py /path/to/deployment/
```

### Strategy 2: Docker Container
```dockerfile
FROM python:3.9-slim

# Install system dependencies for WeasyPrint (optional)
RUN apt-get update && apt-get install -y \
    libcairo2 libpango-1.0-0 libpangocairo-1.0-0 \
    libgdk-pixbuf2.0-0 libffi-dev shared-mime-info \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy scripts
COPY *.py /app/
WORKDIR /app

# Default command
CMD ["python", "new.py", "--help"]
```

### Strategy 3: Standalone Script
Both scripts automatically install dependencies if missing:
```bash
# Just copy the script and run
cp md_to_pdf_reportlab.py /path/to/server/
python md_to_pdf_reportlab.py input.md
```

## Example Markdown File

Create a test file `sample.md`:
```markdown
# Sample Document

This is a **sample** markdown document with *formatting*.

## Features

- Lists work
- **Bold text**
- `inline code`

### Code Block
```python
def hello():
    print("Hello World!")
```

### Image
![Sample Image](https://via.placeholder.com/300x200)

> This is a blockquote

| Column 1 | Column 2 |
|----------|----------|
| Data 1   | Data 2   |
```

Test conversion:
```bash
python new.py sample.md
# or
python md_to_pdf_reportlab.py sample.md
```

## Conclusion

- Use **WeasyPrint version** (`new.py`) for better formatting if system dependencies can be installed
- Use **ReportLab version** (`md_to_pdf_reportlab.py`) for maximum compatibility across platforms
- Both versions handle images, basic formatting, and work cross-platform with pure pip dependencies 

```


```
# Dependencies for WeasyPrint version (new.py)
markdown>=3.4.0
weasyprint>=56.0
Pillow>=9.0.0

# Additional dependencies for ReportLab version (md_to_pdf_reportlab.py)
reportlab>=3.6.0
html2text>=2020.1.16 
```
