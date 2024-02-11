# epub_ingestion_python

e.g. Use epub books from Humble Bunbles in Retrieval Augmented Generation systems with Foundation Models.

```python
#########################################################
# This code automaticaly finds and processes epub books
# esspecially for RAG document ingestion processing
#########################################################
"""
# Set of Results, saved in a file per epub doc:
1. One .jsonl file
2. (Plural) Individual .json files in a folder
3. One .txt text file of the whole epub
4. (plural) Individual .txt files from separate parts of epub
5. chunks as one .jsonl file
6. (Plural) chunks under specified character length as separate text
Future Feature:
7. Chunk-Metadata (for model training, for DB-retrieval, etc.)
"""

import zipfile
import xml.etree.ElementTree as ET
from bs4 import BeautifulSoup
import json
import os
import shutil


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


def extract_text_from_html(html_content, this_epub_output_dir_path):
    """
    Extracts and returns text from an HTML content.
    """
    # print("HTML Content before BeautifulSoup Parsing:\n", html_content[:500])  # Print first 500 characters of HTML
    print_and_log(f"\nlen(HTML Content before BeautifulSoup Parsing) -> {len(html_content)}", this_epub_output_dir_path)  # Print first 500 characters of HTML


    soup = BeautifulSoup(html_content, 'html.parser')
    parsed_text = soup.get_text()
    # print("Extracted Text:\n", parsed_text[:500])  # Print first 500 characters of extracted text
    print_and_log(f"len(Extracted Text) -> {len(parsed_text)}", this_epub_output_dir_path)  # Print first 500 characters of extracted text

    return parsed_text

def fix_text_formatting(text):
    """Replaces the Unicode right single quotation mark with a standard apostrophe."""
    return text.replace("\u2019", "'")


def check_len_chunks_in_list(chunks_list, max_chunk_size, this_epub_output_dir_path):

    size_flag_ok = True

    for index, this_chunk in enumerate(chunks_list):

        # get size of chunk
        this_length = len( this_chunk )

        # check size against user-input max size
        if this_length > max_chunk_size:
            print_and_log( this_length, this_epub_output_dir_path )
            print_and_log( f"""
            Warning: chunk over max size. 
            This chunk size: {this_chunk}. 
            Max size: {max_chunk_size}
            """ )
            print_and_log( f"This chunk: {this_chunk}", this_epub_output_dir_path )
            size_flag_ok = False

    # report and log
    if size_flag_ok:
        print_and_log( "Size Check, OK \o/", this_epub_output_dir_path )
    else:
        print_and_log( "WARNING: Size Check Failed!", this_epub_output_dir_path )



def save_individual_chunks(chunks_list, output_chunks_dir, chunk_source_name):

    for index, this_chunk in enumerate(chunks_list):
        chunk_name = f"{chunk_source_name}_{index}.txt"

        # Save individual txt files
        individual_chunk_path = os.path.join(output_chunks_dir, chunk_name)
        with open(individual_chunk_path, 'w') as f:
            f.write(this_chunk)
            # print('chunk writen', individual_chunk_path)

    return len(chunks_list)


def append_chunks_to_jsonl(chunks_list, output_chunks_jsonl_path, chunk_source_name):
    """Appends chunks of text to a .jsonl file, each chunk as a JSON object.

    Args:
        chunks_list (list): List of text chunks to be appended.
        output_jsonl_path (str): The output file path for the .jsonl file.
        chunk_source_name (str): Base name for each chunk, used in the 'source_name' field.

    Returns:
        int: The number of chunks appended.
    """

    with open(output_chunks_jsonl_path, 'a') as f:  # Open file in append mode
        for index, this_chunk in enumerate(chunks_list):
            # Construct a JSON object for the chunk
            chunk_data = {
                "source_name": f"{chunk_source_name}_{index}",
                "text": this_chunk
            }

            # Convert the chunk data to a JSON string and append it to the file with a newline
            f.write(json.dumps(chunk_data) + '\n')

    return len(chunks_list)


def print_and_log(input_text, this_epub_output_dir_path):
    # check if input is a string, if not...make it a string! 
    if not isinstance(input_text, str):
        input_text = str(input_text)

    # print to terminal
    print(input_text)

    # log file path is...
    log_file_path = os.path.join(this_epub_output_dir_path, "log.txt")

    # log: Write/Append to a log.txt file
    with open(log_file_path, 'a') as f:
        f.write(input_text + '\n\n')


def extract_text_from_epub(epub_file_path, this_epub_output_dir_path, output_jsonl_path, output_json_dir, output_whole_txt_path, output_txt_dir, output_chunks_jsonl_path, output_chunks_dir, max_chunk_size=500):
    with zipfile.ZipFile(epub_file_path, 'r') as epub:
        print_and_log(f"EPUB Contents: -> {epub.namelist()}", this_epub_output_dir_path)

        ###################
        # Make Directories
        ###################

        # Create a directory for individual JSON files
        if not os.path.exists(output_json_dir):
            os.makedirs(output_json_dir)

        # Create a directory for individual txt files
        if not os.path.exists(output_txt_dir):
            os.makedirs(output_txt_dir)

        # Create a directory for chunks output_chunks_dir
        if not os.path.exists(output_chunks_dir):
            os.makedirs(output_chunks_dir)


        ##################################
        # Get & Read html files from epub
        ##################################
        # find opf file
        opf_file = [f for f in epub.namelist() if 'content.opf' in f][0]

        # read opf file
        opf_content = epub.read(opf_file).decode('utf-8')

        # get ordered HTML files
        ordered_html_files_list = get_ordered_html_files(opf_content)


        ############################################
        # Read and extract text from each HTML file
        ############################################

        # iterate through html files
        for html_file in ordered_html_files_list:
            full_path = os.path.join(os.path.dirname(opf_file), html_file)
            if full_path in epub.namelist():
                html_content = epub.read(full_path).decode('utf-8')

                #########################
                # extract text from epub
                #########################
                raw_text = extract_text_from_html(html_content, this_epub_output_dir_path)
                print_and_log(f"len(text for json)-> {len(raw_text)}", this_epub_output_dir_path)

                # fix text formatting
                text = fix_text_formatting(raw_text)


                #################
                # .json & .jsonl
                #################

                # Write/Append to a single JSONL file
                with open(output_jsonl_path, 'a') as f:
                    json_record = json.dumps({'text': text.strip()})
                    f.write(json_record + '\n')

                # Save individual JSON file
                individual_json_path = os.path.join(output_json_dir, f"{os.path.splitext(html_file)[0]}.json")
                with open(individual_json_path, 'w') as f:
                    json.dump({'text': text.strip()}, f, indent=4)

                #######
                # .txt
                #######

                # Write/Append to a single text .txt file
                with open(output_whole_txt_path, 'a') as f:
                    f.write(text + '\n\n')

                # Save individual txt files
                individual_txt_path = os.path.join(output_txt_dir, f"{os.path.splitext(html_file)[0]}.txt")
                with open(individual_txt_path, 'w') as f:
                    f.write(text)

                #########
                # chunks
                #########

                chunks_list = make_chunk_list(text, max_chunk_size, this_epub_output_dir_path)

                chunk_source_name = os.path.splitext(html_file)[0]

                # check sizes
                check_len_chunks_in_list(chunks_list, max_chunk_size, this_epub_output_dir_path)

                number_of_chunks = save_individual_chunks(chunks_list, output_chunks_dir, chunk_source_name)
                print_and_log(f"Chunked: split into this many chunks-> {number_of_chunks}", this_epub_output_dir_path)

                append_chunks_to_jsonl(chunks_list, output_chunks_jsonl_path, chunk_source_name)

                print_and_log(f"{html_file} -> ok!", this_epub_output_dir_path)


            else:  # File Not Found
                print_and_log(f"Warning: File {full_path} not found in the archive.")


def zip_folder(path_to_directory_to_zip, output_destination_zip_file_path):
    """Creates a zip archive of a specified folder.

    Args:
        path_to_directory_to_zip (str): The path to the folder to be zipped.
        output_destination_zip_file_path (str): The desired name and path of the output zip file.
    """
    # # Specify the folder you want to zip
    # path_to_directory_to_zip = "individual_jsons"

    # # Specify the desired output zip file name (e.g., 'jsons_archive.zip')
    # output_destination_zip_file_path = "jsons_archive_zip"

    shutil.make_archive(output_destination_zip_file_path, 'zip', path_to_directory_to_zip)


###########
# Chunking
############

import re

def split_sentences_and_punctuation(text):
    """Splits text into sentences, attempting to preserve punctuation and all text content.

    Args:
        text (str): The input text.

    Returns:
        list: A list of sentences with preserved punctuation.
    """

    # This pattern attempts to split at sentence endings (.?!), including the punctuation with the preceding sentence
    # It uses a lookahead to keep the punctuation with the sentence
    sentence_end_regex = r'(?<=[.!?])\s+(?=[A-Z])'

    split_sentences_and_punctuation_list = re.split(sentence_end_regex, text)

    # Optionally, remove empty strings if they are not desired
    split_sentences_and_punctuation_list = [s for s in split_sentences_and_punctuation_list if s]

    # print("split_sentences_and_punctuation_list")
    # print(split_sentences_and_punctuation_list)

    return split_sentences_and_punctuation_list


def recombine_punctuation(sentences):
    """
    A helper function a that
    Recombines floating punctuation and
    creates a new list of sentences."""
    recombined_sentences = []
    i = 0

    while i < len(sentences) - 1:
        # print(i)
        # print(sentences[i-1])
        # print(sentences[i])
        # print(sentences[i+1])

        sentence = sentences[i].strip()
        next_item = sentences[i + 1].strip()

        # if next_item in ".?!":
        if re.match(r"[.?!]+", next_item):
            recombined_sentences.append(sentence + next_item)
            i += 1  # Skip the punctuation since it's been combined
        else:
            recombined_sentences.append(sentence)
        i += 1

    # Add the last sentence (if it exists)
    if sentences[-1]:
        recombined_sentences.append(sentences[-1].strip())

    return recombined_sentences


def chunk_text(sentences, chunk_size):
    chunked_text = []
    current_chunk = ""

    for sentence in sentences:
        # Case 1: Chunk + sentence easily fit
        if len(current_chunk) + len(sentence) + 1 <= chunk_size:
            current_chunk += sentence + " "

        # Case 2: Sentence itself is too big
        elif len(sentence) > chunk_size:
            # Split long sentence (implement 'split_long_sentence' below)
            for sub_sentence in split_long_sentence(sentence, chunk_size):
                chunked_text.append(sub_sentence.strip())

        # Case 3:  Chunk + sentence exceed limit, time to split
        else:
            chunked_text.append(current_chunk.strip())
            current_chunk = sentence + " "

    # Handle final chunk
    if current_chunk:
        chunked_text.append(current_chunk.strip())

    return chunked_text


def split_long_sentence(sentence, chunk_size):
    """Splits a long sentence into chunks, aiming near the  chunk_size."""
    words = sentence.split()
    chunks = []
    current_chunk = ""

    for word in words:
        if len(current_chunk) + len(word) + 1 <= chunk_size:
            current_chunk += word + " "
        else:
            chunks.append(current_chunk.strip())
            current_chunk = word + " "

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks


def check_for_not(chunk, window_size=25):
    """Checks if 'not' is isolated near a potential cut.

    Args:
        chunk (list): A list of sentences forming the chunk.
        window_size (int): The number of characters to consider on either side.

    Returns:
        bool: True if 'not' is isolated, False otherwise.
    """

    joined_chunk = ' '.join(chunk)
    not_indices = [m.start() for m in re.finditer(r'\bnot\b', joined_chunk)]

    for index in not_indices:
        start = max(0, index - window_size)
        end = min(len(joined_chunk), index + window_size)
        if not re.search(r'\w', joined_chunk[start:end]):  # Check for surrounding words
            return True

    return False


def make_chunk_list(text, chunk_size, this_epub_output_dir_path):
    split_sentences_list = split_sentences_and_punctuation(text)
    chunk_list = chunk_text(split_sentences_list, chunk_size)

    for i in chunk_list:
        if not i:
            print_and_log("error None in chunk_list: make_chunk_list()")

    print_and_log(f"len chunk list -> {len(chunk_list)}", this_epub_output_dir_path)

    return chunk_list


######
# Run
######
"""
1. add your epub files into the same current working directory as this script
2. run script
3. find the files in new folders per epub
"""
import glob

# Get the current working directory.
cwd = os.getcwd()

# Search for all EPUB files in the current working directory.
epub_files = glob.glob(os.path.join(cwd, "*.epub"))

# Print the list of EPUB files.
print(epub_files)


####################
# run for each epub
####################
for this_epub_file in epub_files:

    # set target epub to first epub doc listed as being in the cwd
    epub_file_path = this_epub_file

    # make directory for this book
    this_epub_output_dir_path = epub_file_path[:-5] + "_epub_folder"
    print(this_epub_output_dir_path)

    # Set the absolute path
    this_epub_output_dir_path = os.path.abspath(this_epub_output_dir_path)

    # Create a directory for individual txt files
    if not os.path.exists(this_epub_output_dir_path):
        os.makedirs(this_epub_output_dir_path)

    # json
    # output_jsonl_path = 'output.jsonl'
    output_jsonl_path = os.path.join(this_epub_output_dir_path, 'output.jsonl')
    output_json_dir = os.path.join(this_epub_output_dir_path, 'individual_jsons')  # Directory to store individual JSON files
    output_json_zip_dir = os.path.join(this_epub_output_dir_path, 'jsons_zip_archive')  # Directory to store individual JSON files

    # txt
    output_whole_txt_path = os.path.join(this_epub_output_dir_path, 'whole.txt')
    output_txt_dir = os.path.join(this_epub_output_dir_path, 'individual_txt')  # Directory to store individual txt files
    output_txt_zip_dir = os.path.join(this_epub_output_dir_path, 'txt_zip_archive')  # Directory to store individual JSON files


    # chunks
    output_chunks_dir = os.path.join(this_epub_output_dir_path, 'chunk_text_files')  # Directory to store individual txt files
    output_chunks_jsonl_path = os.path.join(this_epub_output_dir_path, 'chunks_jsonl_all')  # Directory to store individual txt files
    output_chunks_zip_dir = os.path.join(this_epub_output_dir_path, 'chunks_zip_archive')  # Directory to store individual JSON files

    extract_text_from_epub(epub_file_path,
                           this_epub_output_dir_path, 
                           output_jsonl_path, 
                           output_json_dir, 
                           output_whole_txt_path, 
                           output_txt_dir, 
                           output_chunks_jsonl_path, 
                           output_chunks_dir, 
                           max_chunk_size=500)


    # Call the zip function
    """
    zip_folder(path_to_directory_to_zip, output_destination_zip_file_path)
    """
    zip_folder(output_json_dir, output_json_zip_dir)
    zip_folder(output_txt_dir, output_txt_zip_dir)
    zip_folder(output_chunks_dir, output_chunks_zip_dir)



```
