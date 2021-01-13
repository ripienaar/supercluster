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
# TC - enables Linux traffic control shaping, only on Linux servers
# JETSTREAM - enables JetStream on all servers in the first cluster

def parse_env!
  @domain = ENV.fetch("DOMAIN", "example.net")
  @image = ENV.fetch("IMAGE", "nats")
  @image_config = ENV.fetch("IMAGE_CONFIG", "/nats-server.conf")
  @password = ENV["PASSWORD"]
  @tc = ["yes", "true", "1"].include?(ENV["TC"])

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

  if ENV.include?("JETSTREAM")
    abort("JetStream is only supported in single cluster deploys") if @clusters > 1
    @jetstream = true
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

def leafnode_port(cluster, node)
  abort("Only %d clusters are defined" % @clusters) if cluster > @clusters
  abort("Only %d cluster members are defined" % @cluster_members) if node > @cluster_members

  @base_port + ((cluster-1)*@cluster_members) + node-1 + 100
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

  (1..@clusters).each do |cluster|
    network = "nats-cluster%d" % cluster
    networks[network] = {}
    cluster_domain = "c%d.%s" % [cluster, @domain]

    puts "Cluster: %d" % cluster

    (1..@cluster_members).each do |node|
      name = "nc%d.%s" % [node, cluster_domain]
      port = node_port(cluster, node)
      lport = leafnode_port(cluster, node)
      container_name = "nc%d-c%s" % [node, cluster]

      puts "  Node: %s" % name
      puts "         Port: %d" % port
      puts "    Leaf Port: %d" % lport
      puts

      services[name] = {
        "container_name" => container_name,
        "image" => @image,
        "dns_search" => cluster_domain,
        "labels" => {
          "com.docker-tc.enabled" => "1",
        },
        "environment" => {
          "GATEWAY" => "c%d" % cluster,
          "NAME" => container_name,
          "ADVERTISE" => "localhost:%d" % port
        },
        "networks" => [
          network,
          "shared"
        ],
        "volumes" => [
          "./configs/cluster.conf:%s" % @image_config
        ],
        "ports" => [
          "%d:4222" % port,
          "%d:7422" % lport
        ]
      }
    end

    puts
  end

  if @tc
    services["tc.%s" % @domain] = {
      "container_name" => "traffic-control",
      "image" => "lukaszlach/docker-tc",
      "dns_search" => @domain,
      "network_mode" => "host",
      "cap_add" => ["NET_ADMIN"],
      "volumes" => [
        "/var/run/docker.sock:/var/run/docker.sock",
      ],
      "environment" => {
        "HTTP_BIND" => "0.0.0.0",
        "HTTP_PORT" => "4080"
      }
    }
  end

  composev3
end

def start_shell
  abort "Shell not started, node and cluster could not be determined" unless @shell_node && @shell_cluster

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

def config
  ERB.new(File.read("configs/cluster.erb"), nil, "-").result(binding)
end

desc "Starts a shell"
task :shell do
  parse_env!

  start_shell
end

desc "Creates a docker-compose based super cluster"
task :supercluster do
  parse_env!

  File.open("docker-compose.yaml", "w") do |f|
    f.print compose.to_yaml
  end
  puts "...wrote docker-compose.yaml"

  File.open("configs/cluster.conf", "w") do |f|
    f.print config
  end
  puts "...wrote configs/cluster.conf"

  puts
  puts "Use rake shell to access the network"
end
