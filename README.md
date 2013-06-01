# redmine-gource

Redmine activity visualization using [gource](http://code.google.com/p/gource/).

## Legend

* Level 1 - Projects and subprojects.

* Level 2 - Assignies

* Level 3 - Issues

## Usage

* install gource:

        sudo apt-get install gource

* Setup connection to redmine database:

    cp config.rb.example config.rb
    vi config.rb

* Extract data from redmine database:

        rake extract > gource.log

* Download gravatars for all redmine users:

        rake gravatars

* Enjoy:

        gource --user-image-dir gravatars gource.log

## Authors

[Dmitriy Pashin](dmitriy.pashin@yandex.ru)

## License

[WTFPL](http://www.wtfpl.net/)
