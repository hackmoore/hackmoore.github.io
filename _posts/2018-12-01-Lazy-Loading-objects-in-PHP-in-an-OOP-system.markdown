---
title: Lazy Loading objects in PHP in an OOP system
description: How drastically reduced the loading speed, number of database queries and memory usage of my system.
categories:
 - programming
tags:
 - hackthebox
 - guide
---

## Notes
The code for this project is used in a production system and not owned by me and therefore I cannot include large sections of the code but more the ideas and smaller snippets.

## The problem
While working on a project, I followed a very heavy OOP approach to development. The system inheritly supported a OOP structure which made this easy. Using a smaller data set the code ran without any issues. I had something similar to this set up.

```php
<?php
    class objectTemplate{
        // Standard elements
        public $id;
        public $title;
        protected $fields = array();
        public $dbResults;

        protected function __construct($id){
            $this->id = $id;

            $this->dbResults = getDetails();

            // Copy in the results into the fields
            foreach($this->fields as $field)
                $this->$field = $this->dbResults[$field];
        }

        public function get($field){
            return $this->$field;
        }
    }

    class person extends objectTemplate{
        protected $fields = array('firstname', 'lastname');

        public $firstname;
        public $lastname;
    }

    class item extends objectTemplate{
        public $author;

        public function __construct(){
            parent::__construct();
            $this->author = new person($this->dbResults['author']);
        }
    }

    class collection extends objectTemplate{
        public $items;

        public function __construct(){
            parent::__construct();

            // Foreach item, create an instance of that item
            foreach($this->dbResults['items'] as $itemid){
                array_push($this->items, new item($itemid));
            }
        }
    }
?>
```

As you can likely follow, this code works well and provides the advantage of being able to do something like this

```php
<?php
    // Get collection id=1
    $collection = new collection(1);

    // Get the firstname of the author of the second item in the first collection
    $author = $collection->get('items')[1]->author->get('firstname');
?>
```

One problem that I did find when running something like this in a larger dataset is that the server is making a large amount of requests and burning through memory. Due to the larger number of requests it also was taking some time to respond to the requests.


## The solution
This required the Lazy Loading design patern which focuses on initalising objects in a JIT ("Just In Time") method. This means that objects are created as a shell and then information is loadinged into them on a need by need basis.

This ended up looking like this.

```php
<?php
    class objectTemplate{
        // Standard elements
        public $id;
        public $title;
        protected $fields = array();
        public $dbResults;

        protected function __construct($id){
            $this->id = $id;
        }

        public function loadData(){
            if( is_null($this->dbResults) )
                $this->dbResults = getDetails();

            // Copy in the results into the fields
            foreach($this->fields as $field)
                $this->$field = $this->dbResults[$field];
        }

        public function get($field){
            if( $field != 'id' )
                $this->loadData();
            return $this->$field;
        }
    }
?>
```