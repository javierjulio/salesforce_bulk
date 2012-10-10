# SalesforceBulk

## Overview

SalesforceBulk is an easy to use Ruby gem for connecting to and using the [Salesforce Bulk API](http://www.salesforce.com/us/developer/docs/api_asynch/index.htm). This is a rewrite and separate release of Jorge Valdivia's salesforce_bulk gem (renamed `salesforcebulk`) with full unit tests and full API capability (e.g. adding multiple batches per job). This gem was built on Ruby 1.8.7, 1.9.2, and 1.9.3.

## Installation

Install SalesforceBulk from RubyGems:

    gem install salesforcebulk

Or include it in your project's `Gemfile` with Bundler:

    gem 'salesforcebulk'

## Contribute

To contribute, fork this repo, create a topic branch, make changes, then send a pull request. Pull requests without accompanying tests will *not* be accepted. If you need help creating tests let me know and I'll help out. To setup the project and run tests in your fork, just do:

    bundle install
    rake

## Configuration and Initialization

### Basic Configuration

When retrieving a password you will also be given a security token. Combine the two into a single value as the API treats this as your real password.

    require 'salesforce_bulk'
    
    client = SalesforceBulk::Client.new(username: 'MyUsername', password: 'MyPasswordWithSecurtyToken')
    client.authenticate

Optional keys include `login_host` (default is 'login.salesforce.com') and `version` (default is '24.0').

### Configuring from a YAML file

Create a YAML file with the content below. Only `username` and `password` is required.

    ---
    username: MyUsername
    password: MyPassword
    login_host: login.salesforce.com # default
    version: 24.0 # default

Then in a Ruby script:

    require 'salesforce_bulk'
    
    client = SalesforceBulk::Client.new("config/salesforce_bulk.yml")
    client.authenticate

## Usage Examples

An important note about the data in any of the examples below: each hash in a data set must have the same set of keys. If you need to have logic to not include certain values simply specify a nil value for a key rather than not including the key-value pair.

### Basic Overall Example

    data1 = [{:Name__c => 'Test 1'}, {:Name__c => 'Test 2'}]
    data2 = [{:Name__c => 'Test 3'}, {:Name__c => 'Test 4'}]
    
    job = client.add_job(:insert, :MyObject__c)
    
    # easily add multiple batches to a job
    batch = client.add_batch(job.id, data1)
    batch = client.add_batch(job.id, data2)
    
    job = client.close_job(job.id) # or use the abort_job(id) method

### Adding a Job

When adding a job you can specify the following operations for the first argument:
- :delete
- :insert
- :update
- :upsert
- :query

When using the :upsert operation you must specify an external ID field name:

    job = client.add_job(:upsert, :MyObject__c, :external_id_field_name => :MyId__c)

For any operation you should be able to specify a concurrency mode. The default is `Parallel`. The only other choice is `Serial`.

    job = client.add_job(:upsert, :MyObject__c, :concurrency_mode => :Serial, :external_id_field_name => :MyId__c)

### Retrieving Job Information (e.g. Status)

The Job object has various properties such as status, created time, number of completed and failed batches and various other values.

    job = client.job_info(jobId) # returns a Job object
    
    puts "Job #{job.id} is closed." if job.closed? # other: open?, aborted?

### Retrieving Info for a single Batch

The Batch object has various properties such as status, created time, number of processed and failed records and various other values.

    batch = client.batch_info(jobId, batchId) # returns a Batch object
    
    puts "Batch #{batch.id} is in progress." if batch.in_progress?

### Retrieving Info for all Batches

    batches = client.batch_info_list(jobId) # returns an Array of Batch objects
    
    batches.each do |batch|
      puts "Batch #{batch.id} completed." if batch.completed? # other: failed?, in_progress?, queued?
    end

### Retrieving Batch Results (for Delete, Insert, Update and Upsert)

To verify that a batch completed successfully or failed call the `batch_info` or `batch_info_list` methods first, otherwise if you call `batch_result` without verifying and the batch failed the method will raise an error.

The object returned from the following example only applies to the operations: `delete`, `insert`, `update` and `upsert`. Query results are handled differently.

    results = client.batch_result(jobId, batchId) # returns an Array of BatchResult objects
    
    results.each do |result|
      puts "Item #{result.id} had an error of: #{result.error}" if result.error?
    end

### Retrieving Query based Batch Results

To verify that a batch completed successfully or failed call the `batch_info` or `batch_info_list` methods first, otherwise if you call `batch_result` without verifying and the batch failed the method will raise an error.

Query results are handled differently as its possible that a single batch could return multiple results if objects returned are large enough. Note: I haven't been able to replicate this behavior but in a fork by @WWJacob has [discovered that multiple results can be returned](https://github.com/WWJacob/salesforce_bulk/commit/8f9e68c390230e885823e45cd2616ac3159697ef).

    # returns a QueryResultCollection object (an Array)
    results = client.batch_result(jobId, batchId)
    
    while results.any?
      
      # Assuming query was: SELECT Id, Name, CustomField__c FROM Account
      results.each do |result|
        puts result[:Id], result[:Name], result[:CustomField__c]
      end
      
      puts "Another set is available." if results.next?
      
      results.next
      
    end

Note: By reviewing the API docs and response format my understanding was that the API would return multiple results sets for a single batch if the query was to large but this does not seem to be the case in my live testing. It seems to be capped at 10000 records (as it when inserting data) but I haven't been able to verify through the documentation. If you know anything about that your input is appreciated. In the meantime the gem was built to support multiple result sets for a query batch but seems that will change which will simplify that method.

## Contribution Suggestions/Ideas

- Support for Ruby 1.8.7 and/or other platforms
- Rdocs

## Version History

**1.1.0** (August 20, 2012)

* Added travis setup. Support for Ruby 1.9.2 and 1.9.3 specified.
* Removed `token` property on Client object. Specify token in `password` field.
* Accepted pull request for 1.9.3 improvements.
* Description updates in README.

**1.0.0** (August 17, 2012)

* Initial public release.

## License

(The MIT license)

Copyright (c) 2012 Javier Julio

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
