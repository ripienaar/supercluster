require "pp"
require "yaml"
require "erb"
require "json"

# DOMAIN - the domain to create containers in (example.net)
# IMAGE - the docker image to use (nats)
# CLUSTERS - how many clusters to create (2)
# CLUSTER_MEMBERS - how many members in each cluster (3)
# PORT - the base port to start adding each node to (10000)
# IMAGE_CONFIG - the path in the image to use for the configuration (/nats-server.conf)
# PASSWORD - set this to enable multiple accounts using this password
# NATS - opens a shell after creating the configurations for a specific cluster and host. Set to '1 1' for cluster 1 node 1
# TOXICLUSTER - if set to 1 configures toxiproxy between clusters

def parse_env!
  @domain = ENV.fetch("DOMAIN", "example.net")
  @image = ENV.fetch("IMAGE", "nats")
  @image_config = ENV.fetch("IMAGE_CONFIG", "/nats-server.conf")
  @password = ENV["PASSWORD"]
  @toxicluster = ENV["TOXICLUSTER"] == "1"

  begin
    @clusters = Integer(ENV.fetch("CLUSTERS", 2))
  rescue
    abort("Please set CLUSTERS to an integer: %s" % $!)
  end

  begin
    @cluster_members = Integer(ENV.fetch("CLUSTER_MEMBERS", 3))
  rescue
    abort("Please set CLUSTER_MEMBERS to an integer: %s" % $!)
  end

  begin
    @base_port = Integer(ENV.fetch("PORT", 10000))
  rescue
    abort("Please set PORT to an integer: %s", $!)
  end

  if ENV["NATS"]
    @shell_cluster, @shell_node = ENV["NATS"].split(/ /)

    @shell_node = 1 unless @shell_node

    @shell_cluster = Integer(@shell_cluster) rescue abort("NATS should be set to 2 integers")
    @shell_node = Integer(@shell_node) rescue abort("NATS should be set to 2 integers")
  end
end

def node_port(cluster, node)
  abort("Only %d clusters are defined" % @clusters) if cluster > @clusters
  abort("Only %d cluster members are defined" % @cluster_members) if node > @cluster_members

  @base_port + ((cluster-1)*@cluster_members) + node-1
end

def compose
  services = {}
  networks = {
    "shared" => {}
  }
  composev3 = {
    "version" => "3",
    "services" => services,
    "networks" => networks
  }
  toxiproxy = []

  (1..@clusters).each do |cluster|
    network = "nats-cluster%d" % cluster
    networks[network] = {}
    cluster_domain = "c%d.%s" % [cluster, @domain]

    puts "Cluster: %d" % cluster

    (1..@cluster_members).each do |node|
      name = "nc%d.%s" % [node, cluster_domain]
      port = node_port(cluster, node)
      container_name = "nc%d-c%s" % [node, cluster]

      puts "  Node: %s" % name
      puts "    Port: %d" % port
      puts

      services[name] = {
        "container_name" => container_name,
        "image" => @image,
        "dns_search" => cluster_domain,
        "environment" => {
          "GATEWAY" => "c%d" % cluster,
          "NAME" => container_name,
        },
        "networks" => [
          network,
          "shared"
        ],
        "volumes" => [
          "./configs/cluster.conf:%s" % @image_config
        ],
        "ports" => [
          "%d:4222" % port
        ]
      }
    end

    if @toxicluster
      services["toxiproxy.%s" % @domain] = {
        "container_name" => "toxiproxy",
        "image" => "shopify/toxiproxy",
        "dns_search" => @domain,
        "command" => "-host=0.0.0.0 --config=/toxiproxy.json",
        "networks" => [
          "shared"
        ],
        "volumes" => [
          "./configs/toxiproxy.json:/toxiproxy.json",
        ]
      }
    end

    puts
  end

  composev3
end

def start_shell
  return unless @shell_node && @shell_cluster

  ENV["NATS_URL"] = "localhost:%d" % node_port(@shell_cluster, @shell_node)
  if @password
    ENV["NATS_USER"] = "one"
    ENV["NATS_PASSWORD"] = @password
  end

  puts
  puts "Starting a shell in cluster %d connected to node %d for the 'nats' cli with NATS_URL, NATS_USER and NATS_PASSWORD set" % [@shell_cluster, @shell_node]
  puts
  sh ENV.fetch("SHELL", "bash")
end

def toxiconfig
  toxi = []

  (1..@clusters).each do |cluster|
    (1..@cluster_members).each do |node|
      toxi << {
        "name" => "nc%d_c%d" % [node, cluster],
        "listen" => "0.0.0.0:%d" % [25+node+(cluster*1000)],
        "upstream" => "nc%d.c%d.%s:7222" % [node, cluster, @domain],
        "enabled" => true
      }
    end
  end

  JSON.pretty_generate(toxi)
end

def config
  ERB.new(File.read("configs/cluster.erb"), nil, "-").result(binding)
end

desc "Creates a docker-compose based super cluster"
task :supercluster do
  parse_env!

  File.open("docker-compose.yaml", "w") do |f|
    f.print compose.to_yaml
  end
  puts "...wrote docker-compose.yaml"

  File.open("configs/toxiproxy.json", "w") do |f|
    f.print toxiconfig
  end
  puts "...wrote configs/toxiproxy.json"

  File.open("configs/cluster.conf", "w") do |f|
    f.print config
  end
  puts "...wrote configs/cluster.conf"

  start_shell
end
