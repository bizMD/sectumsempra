require 'sugar'
del = require 'del'
{log} = require 'util'
{watch} = require 'chokidar'
{files} = require 'node-dir'
{exec, spawn} = require 'child_process'
{resolve, sep, dirname, extname, basename} = require 'path'

in_pages = resolve __dirname, 'pages'

option '-l', '--local', 'Use this if your CLI is locally, not globally, installed'

# Generate path information about the file
getNames = (path, ext = '') ->
	dir  = dirname path
	fold = basename dir
	base = basename path, ext
	name = dir + sep + base
	{name: name, dir: dir, base: base, folder: fold}

# Filter through all the files in the views by extension
searchFiles = (ext, fcn) ->
	{base, ext} = ext if typeof ext is 'object'

	files in_pages, (err, files) ->
		files
		.findAll (file) -> ext is extname file
		.findAll (file) ->
			base is basename file, ext unless base?
			true
		.forEach (file) -> fcn getNames file, ext

# Filter for everything except the pattern
searchFilesExcept = (regex, fcn) ->
	files in_pages, (err, files) ->
		files
		.exclude regex
		.forEach (n) -> fcn n

# If a server is up, kill it. Then start up a new one
spinServer = (server) ->
	if server?
		log "Server [pid #{server.pid}] is shutting down"
		process.kill server.pid, 'SIGTERM'
	server = spawn 'node' , ['node_modules/coffee-script/bin/coffee' , 'server.coffee']
	log "Server [pid #{server.pid}] is starting up"
	server.on 'error', (a) -> throw err if err?
	server.stdout.on 'data', (data) -> log data.toString()
	server.stderr.on 'data', (data) -> log data.toString()
	server.on 'exit', -> log "Server [pid #{server.pid}] has shut down"
	server

# Clean all files
task 'clean', 'Clean out all files in pages/', ->
	searchFilesExcept /(template.marko|index.coffee|style.scss)$/, del.sync

# The sass command (ruby implementation) is required for this to work
task 'compile:scss', 'Compile the scss files into css', ->
	searchFiles {base: 'style', ext: '.scss'}, ({name, dir, folder}) ->
		sass_cmd = "sass #{name}.scss:#{dir}#{sep}#{folder}.css --style compressed"
		del.sync "#{dir}#{sep}#{folder}.css"
		exec sass_cmd, (e) -> throw e if e?

# The juice command will inline the CSS into the marko files
task 'compile:juice', 'Compile all the template.marko files with juice', (opt) ->
	{local} = opt
	juice = if local then 'node node_modules/juice/bin/juice' else 'juice'
	searchFiles {base: 'template', ext: '.marko'}, ({name, dir, folder}) ->
		juice_cmd = "#{juice} #{name}.marko #{dir}#{sep}#{folder}.marko"
		del.sync "#{dir}#{sep}#{folder}.marko"
		exec juice_cmd

# The coffeebar command will take all coffee files in a folder
task 'compile:coffeebar', 'Compile coffeescript with coffeebar into js', (opt) ->
	{local} = opt
	coffeebar = if local then 'node node_modules/coffeebar/bin/coffeebar' else 'coffeebar'
	searchFiles {base: 'index', ext: '.coffee'}, ({name, dir, folder}) ->
		coffeebar_cmd = "#{coffeebar} -Mmo #{dir}#{sep}#{folder}.js #{dir}"
		del.sync "#{dir}#{sep}#{folder}.js"
		exec coffeebar_cmd, (e) -> throw e if e?

# The browserify+coffeeify command will bundle everything required
task 'compile:coffeeify', 'Compile coffeescript with coffeeify into js', (opt) ->
	{local} = opt
	browserify = if local then 'node node_modules/browserify/bin/cmd.js' else 'browserify'
	searchFiles {base: 'index', ext: '.coffee'}, ({name, dir, folder}) ->
		browserify_cmd = "#{browserify} -t coffeeify --extension='.coffee' -o #{dir}#{sep}#{folder}.js #{name}.coffee"
		del.sync "#{dir}#{sep}#{folder}.js"
		exec browserify_cmd, (e) -> throw e if e?

# Chain all the compilation 
task 'compile', 'Compile the sass, coffee, and marko', ->
	invoke 'clean'
	invoke 'compile:scss'
	invoke 'compile:juice'
	invoke 'compile:coffeeify'

# On change of the important files, recompile and restart the server
task 'watch', 'Trigger compilation and testing on file change', (opt) ->
	invoke 'compile'
	server = spinServer()

	reboot = (path, task = false) ->
		log "Change detected in #{path}"
		invoke "#{task}"
		server = spinServer server

	page_watcher = watch './pages'
	page_watcher.on 'change', (path) ->
		switch basename path
			when 'style.scss'
				reboot path, 'compile:scss'
			when 'template.marko'
				reboot path, 'compile:juice'
			when 'index.coffee'
				reboot path, 'compile:coffeeify'

	server_watcher = watch './server.coffee'
	server_watcher.on 'change', -> reboot 'server.coffee', '_nothing'

task 'compress', 'Compress the final HTML, JS, and CSS files', (opt) ->
	echo 'WIP'

task '_nothing', 'This task does absolutely nothing, do not use it', ->