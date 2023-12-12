# epub_ingestion_python

e.g. Use epub books from Humble Bunbles in Retrieval Augmented Generation systems with Foundation Models.

```python
#########################################################
# This block automaticaly finds and processes epub books
# esspecially for RAG document ingestion processing
#########################################################

import zipfile
import xml.etree.ElementTree as ET
from bs4 import BeautifulSoup
import json
import os
import glob


def get_ordered_html_files(opf_content):
    """
    Parses the content.opf file to determine the reading order of HTML files in the EPUB.

    The function reads the 'content.opf' file, which contains metadata about the EPUB's structure.
    It identifies the 'spine' element, which lists the reading order of the content documents,
    and the 'manifest' element, which provides the location of these documents.
    The function returns a list of HTML file paths in the order they should be read.

    Args:
    opf_content (str): A string representation of the content.opf file.

    Returns:
    list: An ordered list of HTML file paths as specified in the EPUB's spine.
    """

    # Parse the content.opf XML content
    tree = ET.ElementTree(ET.fromstring(opf_content))
    root = tree.getroot()

    # Define the namespace for the OPF package file
    ns = {'opf': 'http://www.idpf.org/2007/opf'}

    # Find the spine element which indicates the order of the content documents
    spine = root.find('opf:spine', ns)
    itemrefs = spine.findall('opf:itemref', ns)

    # Extract the id references for each item in the spine
    item_ids = [itemref.get('idref') for itemref in itemrefs]

    # Find the manifest element which lists all the content documents
    manifest = root.find('opf:manifest', ns)
    items = manifest.findall('opf:item', ns)

    # Create a dictionary mapping item IDs to their corresponding file paths
    html_files = {item.get('id'): item.get('href') for item in items if item.get('media-type') == 'application/xhtml+xml'}

    # Generate an ordered list of HTML files based on the spine order
    ordered_html_files = [html_files[item_id] for item_id in item_ids if item_id in html_files]

    return ordered_html_files


def extract_text_from_html(html_content):
    """
    Extracts and returns text from an HTML content.
    """
    #print("HTML Content before BeautifulSoup Parsing:\n", html_content[:500])  # Print first 500 characters of HTML
    print(f"\nlen(HTML Content before BeautifulSoup Parsing) -> {len(html_content)}")  # Print first 500 characters of HTML

    soup = BeautifulSoup(html_content, 'html.parser')
    parsed_text = soup.get_text()
    # print("Extracted Text:\n", parsed_text[:500])  # Print first 500 characters of extracted text
    print(f"\nLen(Extracted Text) -> {len(parsed_text)}")  # Print first 500 characters of extracted text

    return parsed_text


def extract_text_from_epub(epub_path, output_jsonl_path, output_json_dir):
    """
    Extracts text from an EPUB file, writes it to a single JSONL file, and creates individual JSON files for each HTML content.

    Args:
    epub_path (str): Path to the EPUB file.
    output_jsonl_path (str): Path for the output JSONL file that will contain all extracted text.
    output_json_dir (str): Directory path to store individual JSON files.
    """

    with zipfile.ZipFile(epub_path, 'r') as epub:
        print("EPUB Contents:", epub.namelist())

        # Locate and read the content.opf file for metadata
        opf_file = [f for f in epub.namelist() if 'content.opf' in f][0]
        opf_content = epub.read(opf_file).decode('utf-8')

        # Get an ordered list of HTML files based on EPUB structure
        ordered_html_files = get_ordered_html_files(opf_content)

        # Create a directory for individual JSON files if it doesn't exist
        if not os.path.exists(output_json_dir):
            os.makedirs(output_json_dir)

        for html_file in ordered_html_files:
            full_path = os.path.join(os.path.dirname(opf_file), html_file)
            if full_path in epub.namelist():
                # Read and extract text from each HTML file
                html_content = epub.read(full_path).decode('utf-8')
                text = extract_text_from_html(html_content)
                print(f"len(text for json)-> {len(text)}")

                # Append the extracted text to a single JSONL file
                with open(output_jsonl_path, 'a') as f:
                    json_record = json.dumps({'text': text.strip()})
                    f.write(json_record + '\n')

                # Create an individual JSON file for each HTML file
                individual_json_path = os.path.join(output_json_dir, f"{os.path.splitext(html_file)[0]}.json")
                with open(individual_json_path, 'w') as f:
                    json.dump({'text': text.strip()}, f, indent=4)

                print(f"{html_file} -> ok!")
            else:
                print(f"Warning: File {full_path} not found in the archive.")


def make_epub_file_list():
    # This will match all files ending in .epub in the current directory
    list_of_epub_files = glob.glob('*.epub')

    # Print the list of .epub files
    for file in list_of_epub_files:
        print(file)

    return list_of_epub_files


# get list of epub files
list_of_epub_files = make_epub_file_list()
print(f"list_of_epub_files -> {list_of_epub_files}")

# Example usage
epub_file_path = list_of_epub_files[0]
!mkdir "data"
output_jsonl_path = 'data/output.jsonl'
output_json_dir = 'individual_jsons' # Directory to store individual JSON files
extract_text_from_epub(epub_file_path, output_jsonl_path, output_json_dir)

```
