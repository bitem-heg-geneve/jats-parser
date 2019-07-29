# JATS parsing
The project contains python scripts parsing the Europe PMC interchange format JATS using the lxml library

See https://jats.nlm.nih.gov/archiving/tag-library/1.0/index.html?attr=article-type

See also https://lxml.de/parsing.html

## How to use

### process_xml.py

Turns a PMC publication file (in XML JATS format) into json file written on the disk and sends some stats on stdout

The *data* diectory contains some XML JATS publication files copied from  Europe PMC for test purpose

The json output is written in the *out* subirectory

**Usage**

Parsing a single file

    python ./process_xml.py ./data/3_Biotech/PMC3339582.xml
    less ./out/33/pmc3339582.json

Parsing a list of files

    find data -name "*.xml" -exec python ./process_xml.py {} \;

### jsonpmc_httpserver.py

Simple http server providing a json version of a publication given a pmcid.

The server listen http requests on port 8088 and responds ro requests like _/parse/pmc/{som pmcid}_ .

The server retrieves the XML JATS version of the corresponding publication by calling the ebi API.

It then uses **process_xml.py** to parse the XML. The response is a json version of the publication.

**Usage**

    python ./jsonpmc_httpserver.py > mylog 2>&1 &
    # wait 5 seconds
    curl http://localhost:8088/parse/pmc/3507161

## How to install the utility

### How to activate python3 environment:

https://wsvincent.com/install-python3-mac/

    # activate:
    source ~/.virtualenvs/myvenv/bin/activate

    # deactivate:
    deactivate

### How to install and use beautiful soup and lxml:

https://www.crummy.com/software/BeautifulSoup/bs4/doc/

    pip install beautifulsoup4e
    pip install lxml

## Notes about the parser

### Main object structure

The parsing of the publication produces a json object. It is a dictionary with the follwing keys:

* _id: the numeric part of the PMC identifier
* pmcid, pmid, doi: publication identifiers when available
* journal, issue, volume, startPage, endPage, medlinePgn, medlineTA: journal infos
* affiliationList, authorList:
* publicationDate, publicationDateAlt
* articleType
* keywords: an array of keywords
* fullTitle
* abstract
* sections: an array of elements with the publication content, the publication title and the abstract are the first two sections in the array (if they exist)

### Section structure

**id**

A string identifying the section generated during the parsing process.
Sections in the input XML can embed each other. But in the generated json they are all siblings, flattened in a sequence (the _sections_ array).
The hierarchical structure of the section and of any of its content is reflected in its id.
For instance, a section with id _1.3.2_ comes just after section with id _1.3.1_ and and both are subparts of section with id _1.3_ in the original XML.

**level**

Reflects the embedding level of the section in the original XML file.
Sections that are children of the _<body>_ element in the XML have level 1.
A section that is a child of another _<section>_ that is a child of the _<body>_ element of the XML will have level 2.

**implicit**

The XML of many publications do not use sections of use paragraphs ( _<p>_ element ) outside of any section just below the _<body>_ element. In this case, we create an _implicit_ section (equivalent to the body) to keep the json structure consistent.
The _title_ and _abstract_ sections are set as implicit.

**label**

The label of a section as given in the original XML. For instance, a section number.
Often empty.

**title**

The title of a section as given in the original XML.
Ofter empty.

**contents**

A list of textual elements that are part of the section.
Like sections, the content elements also have an _id_ reflecting his hierachical position in the XML tree.
The _tag_ is the name of the XML element containing the textual element.
The most common tags are paragraph _p_,figures _fig_ and tables _table_ .
The field _caption_ contains the text for _fig_ and _table_ elements.
The field _text_ contains the text for paragraphs _p_ and other elements.
Other tags (not _p_ nor _fig_ nor _table_) may appear but no particular effort was made to handle their textual content specifically.

### Handling figures

Figures (elements with tag = _fig_) have specific fields:

* label : the figure label as given in the XML
* caption : the figure caption as given in the XML
* graphics, media and pmcid: identifiers allowing to build an URL containing an image of the figure

In XML, _<fig>_ may appear in:

* <body>, <sec> the moest usual cases
* <p> in this case, the paragraph is split in several subparts, the figure becoming one of them
* <fig-group> : is removed from the XML, its tables are handled normally, see Endosc_Int_Open/PMC4423251.xml ,Curr_Health_Sci_J/PMC3945237.xml
* <boxed-text> : IGNORED. Are out of main body text flow: ignored by removing them
* <disp-quote> : uses the default handler if not included in p or sec

### Handling tables

Tables (elements with tag = _table_) have specific fields:

* label : the table label as given in the XML
* caption : the table caption as given in the XML
* tableColumns : the name of the table column as  text
* tableValues : the values of the table cells as text
* graphics, media and pmcid: identifiers allowing to build an URL containing an image of the table
* xml : the XML to be used to display the table in an HTML page

In XML, _<table-wrap>_ may appear in:

* <sec> : section is the most usual container of a table
* <body> : in this case, the body is treated as a section
* <boxed-text>:  is removed from the XML, its tables are handled normally
* <p> in this case, the paragraph is split in several subparts, the table becoming one of them. i.e. 3_Biotech/PMC3324826.xml
* <supplementary-material> is removed from the XML, its tables are handled normally, implicit <fig> elements in it are also handled, see Biomark_Cancer/PMC3122269.nxml , Eur_J_Rheumatol/PMC6267743.nxml , Adv_Appl_Bioinform_Chem/PMC3459542.nxml
* <table-wrap-group> is removed from the XML, its tables are handled normally. Common label and caption tags are dispatched in each table in there. See Front_Genet/PMC3202977.nxml , Hum_Gene_Ther/PMC4442602.nxml ,	Intern_Med/PMC5088533.nxml , J_Hum_Reprod_Sci/PMC2700667.nxml
* <disp-quote> : An extract or extended quoted passage from another work, usually made typographically distinct from surrounding text. The table caption appears as the _text_ value of the content element with tag _disp-quote_(default handler). See
J_Entrep_Educ/PMC5985942.nxml , Qual_Saf_Health_Care/PMC2602740.nxml
* <fig> : IGNORED. A rare case. The table content only appears in the detailed figure popup but not in full text viewer on PMC website, so the table-wrap content ignored in this case. See Pharmaceutics/PMC2997712.nxml

### Encoding

alpha - α is seen as "\u03b1" in shell but as α in browser
The _unidecode_ method used pubmed_oa_parser.py could also be used to turn greek letters and other special ones into their ascii closest equivalent.
