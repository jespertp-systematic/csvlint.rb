[![Build Status](http://img.shields.io/travis/theodi/csvlint.rb.svg)](https://travis-ci.org/theodi/csvlint.rb)
[![Dependency Status](http://img.shields.io/gemnasium/theodi/csvlint.rb.svg)](https://gemnasium.com/theodi/csvlint.rb)
[![Coverage Status](http://img.shields.io/coveralls/theodi/csvlint.rb.svg)](https://coveralls.io/r/theodi/csvlint.rb)
[![License](http://img.shields.io/:license-mit-blue.svg)](http://theodi.mit-license.org)
[![Badges](http://img.shields.io/:badges-5/5-ff6799.svg)](https://github.com/pikesley/badger)

# CSV Lint

A ruby gem to support validating CSV files to check their syntax and contents.

## Installation

Add this line to your application's Gemfile:

    gem 'csvlint'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install csvlint

## Usage

Currently the gem supports retrieving a CSV accessible from a URL, File, or an IO-style object (e.g. StringIO)

	require 'csvlint'
	
	validator = Csvlint::Validator.new( "http://example.org/data.csv" )
	validator = Csvlint::Validator.new( File.new("/path/to/my/data.csv" )
	validator = Csvlint::Validator.new( StringIO.new( my_data_in_a_string ) )

When validating from a URL the range of errors and warnings is wider as the library will also check HTTP headers for 
best practices
	
	#invoke the validation	
	validator.validate
	
	#check validation status
	validator.valid?
	
	#access array of errors, each is an Csvlint::ErrorMessage object
	validator.errors
	
	#access array of warnings
	validator.warnings
	
	#access array of information messages
	validator.info_messages
	
	#get some information about the CSV file that was validated
	validator.encoding
	validator.content_type
	validator.extension
	
	#retrieve HTTP headers from request
	validator.headers

## Controlling CSV Parsing

The validator supports configuration of the [CSV Dialect](http://dataprotocols.org/csv-dialect/) used in a data file. This is specified by 
passing an options hash to the constructor:

    opts = {
    	"header" => true,
    	"delimiter" => ","
    }
	validator = Csvlint::Validator.new( "http://example.org/data.csv", opts )

The options should be a Hash that conforms to the [CSV Dialect](http://dataprotocols.org/csv-dialect/) JSON structure.

While these options configure the parser to correctly process the file, the validator will still raise errors or warnings for CSV 
structure that it considers to be invalid, e.g. a missing header or different delimiters.

Note that the parser will also check for a `header` parameter on the `Content-Type` header returned when fetching a remote CSV file. As 
specified in [RFC 4180](http://www.ietf.org/rfc/rfc4180.txt) the values for this can be `present` and `absent`, e.g:

	Content-Type: text/csv; header=present

## Error Reporting

The validator provides feedback on a validation result using instances of `Csvlint::ErrorMessage`. Errors are divided into errors, warnings and information 
messages. A validation attempt is successful if there are no errors.

Messages provide context including:

* `category` has a symbol that indicates the category or error/warning: `:structure` (well-formedness issues), `:schema` (schema validation), `:context` (publishing metadata, e.g. content type)
* `type` has a symbol that indicates the type of error or warning being reported
* `row` holds the line number of the problem
* `column` holds the column number of the issue
* `content` holds the contents of the row that generated the error or warning

## Errors

The following types of error can be reported:

* `:wrong_content_type` -- content type is not `text/csv`
* `:ragged_rows` -- row has a different number of columns (than the first row in the file)
* `:blank_rows` -- completely empty row, e.g. blank line or a line where all column values are empty
* `:invalid_encoding` -- encoding error when parsing row, e.g. because of invalid characters 
* `:not_found` -- HTTP 404 error when retrieving the data
* `:stray_quote` -- missing or stray quote
* `:unclosed_quote` -- unclosed quoted field
* `:whitespace` -- a quoted column has leading or trailing whitespace
* `:line_breaks` -- line breaks were inconsistent or incorrectly specified
* `:no_header` -- the CSV file doesn't include a header
* `:empty_column_name` -- a column in the CSV header has an empty name
* `:duplicate_column_name` -- a column in the CSV header has a duplicate name

## Warnings

The following types of warning can be reported:

* `:no_encoding` -- the `Content-Type` header returned in the HTTP request does not have a `charset` parameter
* `:encoding` -- the character set is not UTF-8
* `:no_content_type` -- file is being served without a `Content-Type` header
* `:excel` -- no `Content-Type` header and the file extension is `.xls`
* `:check_options` -- CSV file appears to contain only a single column
* `:inconsistent_values` -- inconsistent values in the same column. Reported if <90% of values seem to have same data type (either numeric or alphanumeric including punctuation)

## Information Messages

There are also information messages available:

* `:nonrfc_line_breaks` -- uses non-CRLF line breaks, so doesn't conform to RFC4180.

## Schema Validation

The library supports validating data against a schema. A schema configuration can be provided as a Hash or parsed from JSON. The structure currently 
follows JSON Table Schema with some extensions.

An example schema file is:

	{
		"fields": [
			{ 
				"name": "id", 
			  	"constraints": { "required": true } 
			},
            { 
               	"name": "price", 
               	"constraints": { "required": true, "minLength": 1 } 
            },
            { 
            	"name": "postcode", 
            	"constraints": { 
            		"required": true, 
            		"pattern": "[A-Z]{1,2}[0-9][0-9A-Z]? ?[0-9][A-Z]{2}" 
            	} 
            }
        ]
	}

Parsing and validating with a schema:

	schema = Schema.load_from_json_table(uri)
	validator = Csvlint::Validator.new( "http://example.org/data.csv", nil, schema )

Supported constraints:

* `required` -- there must be a value for this field in every row
* `unique` -- the values in every row should be unique
* `minLength` -- minimum number of characters in the value
* `maxLength` -- maximum number of characters in the value
* `pattern` -- values must match the provided regular expression
* `type` -- specifies an XML Schema data type. Values of the column must be a valid value for that type
* `minimum` -- specify a minimum range for values, the value will be parsed as specified by `type`
* `maximum` -- specify a maximum range for values, the value will be parsed as specified by `type` 

Supported data types (this is still a work in progress):

* Integer -- `http://www.w3.org/2001/XMLSchema#int`
* Float -- `http://www.w3.org/2001/XMLSchema#float`
* Double -- `http://www.w3.org/2001/XMLSchema#double`
* URI -- `http://www.w3.org/2001/XMLSchema#anyURI`
* Boolean -- `http://www.w3.org/2001/XMLSchema#boolean`
* Non Positive Integer -- `http://www.w3.org/2001/XMLSchema#nonPositiveInteger`
* Positive Integer -- `http://www.w3.org/2001/XMLSchema#positiveInteger`
* Non Negative Integer -- `http://www.w3.org/2001/XMLSchema#nonNegativeInteger`
* Negative Integer -- `http://www.w3.org/2001/XMLSchema#negativeInteger`
	
Schema validation provides some additional types of error and warning messages:

* `:missing_value` (error) -- a column marked as `required` in the schema has no value
* `:min_length` (error) -- a column with a `minLength` constraint has a value that is too short
* `:max_length` (error) -- a column with a `maxLength` constraint has a value that is too long
* `:pattern` (error) --  a column with a `pattern` constraint has a value that doesn't match the regular expression
* `:header_name` (warning) -- the header in the CSV has a column name that doesn't match the schema
* `:missing_column` (warning) -- a row in the CSV file has a missing column, that is specified in the schema. This is a warning only, as it may be legitimate
* `:extra_column` (warning) -- a row in the CSV file has extra column.
* `:unique` (error) -- a column with a `unique` constraint contains non-unique values
* `:out_of_range` (error) -- a column with a `minimum` or `maximum` constraint contains a value that is outside of the range

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
