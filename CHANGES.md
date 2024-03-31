## Solution

So my approach was to write a function named `include_one` in `config.c` which solves our given question. I tried to make keep `include_one` function simple the implementation looks like this:
``` c
// sway/sway/config.c
void load_include_one_config(const char * paths, 
struct sway_config * config, struct sway_instance * swaynag) {
  char * p = strdup(paths);
  char * path = strtok(p, " ");
  while (path != NULL) {
    load_include_configs(path, config, swaynag, true);
    path = strtok(NULL, " ");
  }
  goto cleanup;

  cleanup:
    free(p);
}
```
The Implementation is pretty simple as the function arguments are similar to all the other functions. Now we split the path on each `" "`  and pass it to load_include_configs which is what `include` command uses to include all the files in the directory. As we can see the function is called  `load_include_configs`now I have just added a additional argument called flag in `load_include_configs`function its declaration now looks like:
```c 
// sway/include/config.h
void  load_include_configs(const  char  *path, 
struct  sway_config  *config,
struct  swaynag_instance  *swaynag, bool  flag);
```

Now `bool flag` has been added by me now the use of this argument will explained later. For now we can observe that when calling it from `load_include_one_config` the `flag` has been passed as `true`and `flase` if not from our function.

Now I wrote two additional static function which extract and compare the filename provided the full path. Here is what the implementation looks like:
``` c
// sway/sway/config.c
static char * extract_filename(const char * path) {
  char * filename = strrchr(path, '/');
  if (filename = NULL) {
    return NULL;
  }
  return filename;
}

static bool same_file(const char * path1,
  const char * path2) {
  const char * file1 = extract_filename(path1);
  const char * file2 = extract_filename(path2);
  return (strcmp(file1, file2) == 0);
}
```
Implementation of these functions looks pretty simple and understandable the `extract_filename`gets the string after the last occurrence of `/` character and `same_filename` just compare the two filename and return a `boolean`.  

Now comes the most important function of the function which is responsible for adding each file to our `config struct`data-structure. Its called `load_include_config` now the first change we did was adding a `flag` argument to our `load_include_config` function and it's use-case can be easily explained after we look into the updated code for `load_include_config` function.
``` c
// sway/sway/config.c
static bool load_include_config(const char * path,
    const char * parent_dir,
      struct sway_config * config,
      struct swaynag_instance * swaynag, bool flag) {

    // save parent config
    const char * parent_config = config -> current_config_path;

    char * full_path;
    int len = strlen(path);
    if (len >= 1 && path[0] != '/') {
      len = len + strlen(parent_dir) + 2;
      full_path = malloc(len * sizeof(char));
      if (!full_path) {
        sway_log(SWAY_ERROR, "Unable to allocate full path to included config");
        return false;
      }
      snprintf(full_path, len, "%s/%s", parent_dir, path);
    } else {
      full_path = strdup(path);
    }

    char * real_path = realpath(full_path, NULL);
    free(full_path);

    if (real_path == NULL) {
      sway_log(SWAY_DEBUG, "%s not found.", path);
      return false;
    }

    // check if config has already been included
    int j;
    for (j = 0; j < config -> config_chain -> length; ++j) {
      char * old_path = config -> config_chain -> items[j];
      if (strcmp(real_path, old_path) == 0) {
        sway_log(SWAY_DEBUG, "%s already included once, won't be included again.",
          real_path);
        free(real_path);
        return false;
      }
      // check if function is called from include_one
      if (flag == true && same_file(real_path, old_path) == true) {
        sway_log(SWAY_DEBUG, "%s already included once, won't be included again.",
          real_path);
        free(real_path);
        return false;
      }
    }
```
The function is quite big so let's focus on the part that we have added into the function and that is:
``` c
if (flag == true && same_file(real_path, old_path) == true) {

  sway_log(SWAY_DEBUG, "%s already included once, won't be included again.", real_path);
  free(real_path);
  return false;
}
```

As we can see I'm checking if the `flag` is `true` which means simply means checking if our function is been called from `load_include_one_config` or not, if it does then we check if there is already a file with same-name exist in our config data-structure of all files. If it does then we simply break out of the function with return value of false.

This is basically the logical part of the `include_one` command function. Other than these I just added our `include_one` commands to actually be useful both from the config files or command line.
```c
// sway/sway/commands.c
static const struct cmd_handler config_handlers[] = {
	{ "default_orientation", cmd_default_orientation },
	{ "include", cmd_include },
	{ "include_one", cmd_include_one }
	{ "primary_selection", cmd_primary_selection },
	{ "swaybg_command", cmd_swaybg_command },
	{ "swaynag_command", cmd_swaynag_command },
	{ "workspace_layout", cmd_workspace_layout },
	{ "xwayland", cmd_xwayland },
};
```
and I have also created a separate file which defines our `cmd_include_one` function and it's quite similar to what `include` command already have minus the additional `flag` argument that I have added.

```c
#include "sway/commands.h"
#include "sway/config.h"

struct cmd_results * cmd_include_one(int argc, char ** argv) {
  struct cmd_results * error = NULL;
  if ((error = checkarg(argc, "include", EXPECTED_EQUAL_TO, 1))) {
    return error;
  }

  // We don't care if the included config(s) fails to load.
  load_include_one_config(argv[0], config, & config -> swaynag_config_errors);

  return cmd_results_new(CMD_SUCCESS, NULL);
}
```  
There could be few tweaks here and there to make the command accessible I basically track the `include` command and added similar lines of code for `include_one` command.

Unfortunately sway does not uses GitHub action for there CI/CD tests but rather rely on some other sources to perform the test. So can't show you the test results but yea it does compile in my machine.
