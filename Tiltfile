# curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash
load("./tilt/common.Tiltfile", "CFG", "DEBUG")
load("./tilt/utils.Tiltfile", "require_env", "require_tools", "replace_in_file", "rs_files", "cake_path")
load("ext://color", "color")
load("ext://dotenv", "dotenv")

for dotfile in [
    ".env",
    ".env.development",
    ".env.local",  # last one wins
]:
    if os.path.exists(dotfile):
        print(color.green("Loading environment from " + dotfile))
        dotenv(fn = dotfile)
        watch_file(dotfile)

if os.path.exists(".env.example"):
  for line in str(read_file(".env.example")).split("\n"):
      # skip if line is empty or commented out
      if line.find("=") == -1 or line.find("#") != -1:
          continue

      # split key and value
      key, _ = line.split("=", 1)
      print(color.blue("checking for " + key)) if DEBUG else None
      require_env(key)

# Require necessary tools
require_tools("cargo")

# Define the Rust compilation command
def rust_build_cmd():
    return "cargo build --" + CFG.release + " --features " + CFG.features

# Define the Rust run command
def rust_run_cmd(args):
    return "cargo run --bin cake-cli --features " + CFG.features + " -- " + args

cake_src_files = (["Cargo.toml", "Cargo.lock"] +
      rs_files("cake-cli") +
      rs_files("cake-core") +
      rs_files("cake-split-model") +
      rs_files("cake-ios") +
      rs_files("cake-ios-worker-app"))

# Compile the Cake project
local_resource(
    "compile-cake",
    cmd = rust_build_cmd(),
    deps = ["Cargo.toml", "Cargo.lock"] +
      rs_files("cake-cli") +
      rs_files("cake-core") +
      rs_files("cake-split-model") +
      rs_files("cake-ios") +
      rs_files("cake-ios-worker-app"),
    labels = ["build"],
    allow_parallel=True,
    auto_init=False
)

# Function to run a Cake worker
def cake_worker(name, port, worker_index):
    tilt_name = "cake-worker-{name}".format(name = name)
    local_resource(
        name=tilt_name,
        serve_cmd = rust_run_cmd("--model {model_path} --mode worker --name {name} --topology {topology_path} --address 0.0.0.0:{port} {device}".format(
            name=name,
            port=port,
            model_path = CFG.model_path,
            topology_path = CFG.topology_path,
            device = "--device {worker_index}".format(worker_index = worker_index) if CFG.features == "cuda" else "",
        )),
        resource_deps = [
          # "compile-cake"
        ],
        labels = ["worker"],
        deps=[
          CFG.topology_path,
        ] + cake_src_files,
        allow_parallel=True
    )
    return "cake-worker-{name}".format(name = name)

# Run two Cake workers
worker_resources = [
  cake_worker(name="worker1", port=10128, worker_index=0),
  cake_worker(name="worker2", port=10129, worker_index=1)
]

# Run the Cake API
local_resource(
    "cake-api",
    serve_cmd = rust_run_cmd("--model {model_path} --api 0.0.0.0:8080 --topology {topology_path}".format(
        model_path = CFG.model_path,
        topology_path = CFG.topology_path,
    )),
    resource_deps = worker_resources,
    labels = ["api"],
    deps=[
      CFG.topology_path,
    ] + cake_src_files,
    readiness_probe=probe(
      period_secs=5,
      http_get=http_get_action(port=8080, path="/alive"),
    ),
    allow_parallel=True
)
