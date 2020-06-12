def distributedTasks = [:]

stage("Building Distributed Tasks") {
  jsTask {
    checkout scm
    sh 'yarn install'


// To run jobs in parallel with jenkins, we need to construct a map of string -> closure where closure contains the code we want to be running in parallel.

    distributedTasks << distributed('test', 3)
    distributedTasks << distributed('lint', 3)
    distributedTasks << distributed('build', 3)
  }
}

stage("Run Distributed Tasks") {
  parallel distributedTasks
}

def jsTask(Closure cl) {
  node {
    withEnv(["HOME=${workspace}"]) {
      docker.image('node:latest').inside('--tmpfs /.config', cl)
    }
  }
}

def distributed(String target, int bins) {
  def jobs = splitJobs(target, bins)  // [["app1", "app2"], ["app3", "app4"]] 
  def tasks = [:]

  jobs.eachWithIndex { jobRun, i -> // ["app1", "app2"] ["app3", "app4"]
    def list = jobRun.join(',') // "app1,app2"  "app3,app4"
    def title = "${target} - ${i}" // build - 1  build - 2

 // To run jobs in parallel with jenkins, we need to construct a map of string -> closure where closure contains the code we want to be running in parallel.


    tasks[title] = {
      jsTask {
        stage(title) {
          checkout scm
          sh 'yarn install'
          sh "npx nx run-many --target=${target} --projects=${list} --parallel" // list = app1,app2 OR app3,app4
        }
      }
    }
  }

  return tasks // build - 1: {}
               // build - 2: {}
}

def splitJobs(String target, int bins) {
  def String baseSha = env.CHANGE_ID ? 'origin/master' : 'origin/master~1'
  def String raw
  raw = sh(script: "npx nx print-affected --base=${baseSha} --target=${target}", returnStdout: true)
  def data = readJSON(text: raw)

  def tasks = data['tasks'].collect { it['target']['project'] } // Suppose it gives 4 apps to build

  if (tasks.size() == 0) {
    return tasks
  }

  // this has to happen because Math.ceil is not allowed by jenkins sandbox (╯°□°）╯︵ ┻━┻
  def c = sh(script: "echo \$(( ${tasks.size()} / ${bins} ))", returnStdout: true).toInteger()
  def split = tasks.collate(c)

  return split // [["app1", "app2"], ["app3", "app4"]] for bin count 2 indicating 2 build agents may be
}
