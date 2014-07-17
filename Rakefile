require 'pathname'
require 'yaml'
require 'json'
require 'erb'
require 'slim'

class Resume
    attr_accessor :contact, :work, :education, :open_source, :hobbies, :profile, :languages

    def erb_binding
        self.send(:binding)
    end

    def initialize(hash = {})
        self.contact = Contact.new(hash['contact'])
        self.profile = hash['profile']
        self.work = (hash['work'] || []).map {|w| Job.new(w)}
        self.education = (hash['education'] || []).map { |e| School.new(e) }
        self.open_source = (hash['open_source'] || []).map { |o| OpenSource.new(o) }
        self.hobbies = hash['hobbies']
        self.languages = hash['languages']
    end

    class Job
        attr_accessor :name, :website, :location, :position, :start, :end, :accomplishments
        def initialize(hash = {})
            %w{name website location position start end accomplishments}.each do |key|
                self.send("#{key}=", hash[key])
            end
        end

        def to_h
            Hash[%w{name website location position start end accomplishments}.map { |k| [k, self.send(k)] }]
        end
    end

    class School
        attr_accessor :name, :location, :degree, :start, :end, :aside
        def initialize(hash = {})
            %w{name location degree start end aside}.each do |key|
                self.send("#{key}=", hash[key])
            end
        end

        def to_h
            Hash[%w{name location degree start end aside}.map { |k| [k, self.send(k)] }]
        end
    end

    class OpenSource
        attr_accessor :github_url, :name, :organization
        def initialize(string)
            self.organization, self.name = string.split('/')
            self.github_url = "https://github.com/#{organization}/#{name}"
        end

        def to_h
            {
                github_url: github_url,
                name: name,
            }
        end
    end

    class Contact
        attr_accessor :name, :phone, :website, :email, :address
        def initialize(hash = {})
            %w{name phone website email}.each do |key|
                self.send("#{key}=", hash[key])
            end
            self.address = Address.new(hash['address'])
        end

        class Address
            attr_accessor :street, :city, :state, :zip
            def initialize(hash = {})
                %w{street city state zip}.each do |key|
                    self.send("#{key}=", hash[key])
                end
            end

            def to_h
                {
                    street: street,
                    city: city,
                    state: state,
                    zip: zip,
                }
            end
        end

        def to_h
            {
                name: name,
                address: address.to_h,
                phone: phone,
                website: website,
                email: email,
            }
        end
    end

    def to_h
        {
            contact: contact.to_h,
            profile: profile,
            work: work.map(&:to_h),
            education: education.map(&:to_h),
            open_source: open_source.map(&:to_h),
            hobbies: hobbies,
            languages: languages,
        }
    end
end

LATEX_HTML = %~<span style="font-family:cmr10,'Latin Modern Roman',serif">L<span style="text-transform: uppercase; font-size: 70%; margin-left: -0.36em; vertical-align: 0.3em; line-height: 0; margin-right: -0.15em">a</span>T<span style="text-transform: uppercase; margin-left: -0.1667em; vertical-align: -0.5ex; line-height: 0; margin-right: -0.125em">e</span>X</span>~

def escape_html(input='')
  case input
  when Hash
    Hash[input.map{|k,v| [k, escape_html(v)]}]
  when Array
    input.map{|elt| escape_html(elt)}
  when String
    # Do the actual escaping here
    input.gsub(/\[&\]/, "&amp;")         # Ampersands
      .gsub(/\[LaTeX\]/, LATEX_HTML)  # LaTeX
      .gsub(/\[LaTeX \]/, LATEX_HTML) # LaTeX (trailing space)
      .gsub(/\[``\]/, "&ldquo;")      # Smart quotes (open)
      .gsub(/\[''\]/, "&rdquo;")      # Smart quotes (close)
      .gsub(/\[#\]/, "#")             # Number sign
      .gsub(/\[\$\]/, "$")            # Dollar sign
      .gsub(/\[(.*)\]\((.*)\)/, "<a href=\"\\2\">\\1</a>")   # Link handling
      .chomp
      .gsub(/\n/, '<br>')
  else
    input
  end
end

def escape_latex(input='')
  case input
  when Hash
    Hash[input.map{|k,v| [k, escape_latex(v)]}]
  when Array
    input.map{|elt| escape_latex(elt)}
  when String
    # Do the actual escaping here
    input.gsub(/&/, "\\\\&")            # Ampersands
      .gsub(/LaTeX/, "\\\\LaTeX")       # LaTeX
      .gsub(/``/, "``")                 # Smart quotes (open)
      .gsub(/''/, "''")                 # Smart quotes (close)
      .gsub(/#/, "\\\\#")               # Number sign
      .gsub(/\$/, "\\\\$")              # Dollar sign
      .gsub(/\[(.*)\]\((.*)\)/, "\\\\href{\\2}{\\1}")   # Link handling
      .chomp
      .gsub(/\n/, '\\\\\\')
  else
    input
  end
end

desc 'Generate all of the resume formats'
task :generate => 'generate:all'
task :default => :generate

desc "Remove all generated files"
task :clean do
    files = Dir['build/*.{json,md,html,tex,pdf}']
    files.map { |f| Pathname.new(f).expand_path }.each &:delete
end

namespace :generate do
    task :all => [:json, :md, :html, :pdf]

    desc 'Generate the resume LaTeX file'
    task :latex do
        resume = resume(escape_latex(yaml))
        template = Pathname.new('templates/resume.tex.erb').expand_path.open(&:read)
        latex = ERB.new(template).result(resume.erb_binding)
        Pathname.new(filename('tex')).open('w') { |f| f.write latex }
    end

    desc 'Generate the resume markdown file'
    task :md do
        resume = resume(escape_html(yaml))
        template = Pathname.new('templates/resume.md.erb').expand_path.open(&:read)
        md = ERB.new(template).result(resume.erb_binding)
        Pathname.new(filename('md')).open('w') { |f| f.write md }
    end

    desc 'Generate the resume html file'
    task :html do
        resume = resume(escape_html(yaml))
        html = Slim::Template.new('templates/resume.slim').render(resume)
        Pathname.new(filename('html')).open('w') { |f| f.write html }
    end

    desc 'Generate the resume pdf file'
    task :pdf => [:latex] do
        cd build_dir do
            `PATH=/usr/texbin:$PATH /usr/texbin/xelatex -file-line-error -interaction=nonstopmode -synctex=1 #{filename('tex').split('/').last.shellescape}`
            `rm #{Dir.glob('*.{aux,fdb*,out,log,sync*}').map(&:shellescape).join(' ')}`
        end
    end

    desc 'Generate the resume json file'
    task :json do
        Pathname.new(filename('json')).open('w') { |f| f.write JSON.pretty_generate(resume(yaml).to_h) }
    end

    def yaml
        @yaml ||= Pathname.new('resume.yml').expand_path.open {|f| YAML.load(f.read)}
    end

    def resume(hash)
        Resume.new(hash)
    end

    def build_dir
        @build_dir ||= 'build'.tap {|d| `mkdir -p #{d}`}
    end

    def filename(ext = '')
        File.join build_dir, case ext
        when 'pdf', 'tex'
           "#{resume(yaml).contact.name} Resume.#{ext}"
        else
            "resume.#{ext}"
        end
    end
end
