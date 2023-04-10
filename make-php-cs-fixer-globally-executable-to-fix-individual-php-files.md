---
description: >-
  Easily format individual PHP files by making the PHP-CS-Fixer tool globally
  executable, enabling quick and consistent coding style fixes across your
  projects.
---

# Make php-cs-fixer globally executable to fix individual PHP files

1. Create a new file named `phpfix` with the following content:

```shell
#!/bin/bash

# Check if Composer is installed
if ! command -v composer &>/dev/null; then
  # Check if the user has write permission to /usr/local/bin
  if [ ! -w "/usr/local/bin" ]; then
    echo "Error: You do not have enough permission to write to /usr/local/bin."
    echo "Please run this script with sudo or as a user with enough permission."
    exit 1
  fi

  echo "Composer not found. Installing..."
  curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
  echo "Composer installed successfully."
fi

# Check if php-cs-fixer is installed globally
if ! command -v php-cs-fixer &>/dev/null; then
  echo "php-cs-fixer not found. Installing..."
  composer global require friendsofphp/php-cs-fixer
  echo "php-cs-fixer installed successfully."
fi

php_cs_fixer() {
  file_path="$1"
  config_file="${HOME}/.php-cs-fixer.php"

  if [ -z "$file_path" ]; then
    echo "Please provide a PHP file path after the function name."
    echo "To exit, type 'x' or 'q'."
    return
  elif [ "$file_path" == "x" ] || [ "$file_path" == "q" ]; then
    exit 0
  elif [ ! -f "$file_path" ]; then
    echo "Error: File does not exist. Please input a valid file path."
    return
  elif [[ "$file_path" != *.php ]]; then
    echo "Error: Only PHP files are allowed."
    return
  fi

  if [ -f "$config_file" ]; then
    php-cs-fixer fix "$file_path" --allow-risky=yes --config="$config_file"
  else
    php-cs-fixer fix "$file_path" --allow-risky=yes
  fi

  echo "File has been fixed and formatted."
}

while true; do
  echo -n "phpfix> "
  read -ra input_args
  php_cs_fixer "${input_args[@]}"
done
```

2. Save the file and open a `terminal`.
3. Change the permissions of the `phpfix` script to make it executable:

```shell
chmod +x phpfix
```

4. Move the `phpfix` script to `/usr/local/bin` to make it globally executable:

```shell
sudo mv phpfix /usr/local/bin/phpfix
```

Now you can run the `phpfix` command in your `terminal`It will take the user input and perform the desired actions per your requirements.
