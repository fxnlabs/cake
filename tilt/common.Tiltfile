# -*- mode: python -*-

load("ext://color", "color")

DEBUG = os.getenv("DEBUG", "").find("tilt") != -1

CI = bool(os.getenv(
    "CI",
    config.tilt_subcommand == "ci",
))

run_id = str(local(
    "echo $RANDOM | shasum | head -c 10",
    echo_off = True,
    quiet = True,
)).strip()

def get_cfg():
    envcfg = os.getenv("TILT_CONFIGURED_PARSED")
    if envcfg == None:
        _cfg = None
    else:
        _cfg = decode_json(envcfg)
    if _cfg == None or _cfg.get("run_id") != run_id:
        config.define_string_list(
            "args",
            args = True,
        )

        config.define_string("release", False, "The rust build profile to use. Defaults to 'release'.")
        config.define_string("features", False, "The Transformer features to enable.")
        config.define_string("model_path", False, "The path to the model to use. Defaults to './cake-data/Meta-Llama-3-8B/'.")
        config.define_string("topology_path", False, "The path to the node topology file. Defaults to './cake-data/topology.yml'.")

        _cfg = config.parse()

        _cfg["run_id"] = run_id

        config.set_enabled_resources(_cfg.get("args", []))

        os.putenv("TILT_CONFIGURED_PARSED", encode_json(_cfg))
        print(color.blue("Parsed config: ") + encode_json(_cfg)) if DEBUG else None

    return struct(
        args = _cfg.get("args", []),
        release = _cfg.get("release", "release"),
        features = _cfg.get("features", ""),
        model_path = _cfg.get("model_path", "./cake-data/Meta-Llama-3-8B/"),
        topology_path = _cfg.get("topology_path", "./cake-data/topology.yml"),
    )

CFG = get_cfg()

