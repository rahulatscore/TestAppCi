
# Util =========================================================================
# returns the list of files modified in this PR
#
require 'oga'

slack.api_token = "xoxp-9573735638-182414543844-241586307878-e2dfae0b4833fd3cc0854792434d3db1"

TEAM_MAPPING = {
    'vkondrav' => 'vitaliy.kondratiev',
    'jaregier' => 'jordan.regier',
    'erosales-the-score' => 'edward.rosales',
    'MattPflance' => 'matthew.pflance',
    'bagarwal18' => 'bhavna.agarwal',
    'rahulatscore' => 'rahul.patel'
}

def detekt_report_file
      return 'app/build/reports/detekt-checkstyle.xml'
end

def findbugs_report_file
      return 'app/build/reports/findbugs/findbugs.html'
end

def pr_report()
    message("Delete branch after merging :wastebasket:")

    addPRLabel('needs-review')

    if github.branch_for_base == "internal"
        add_internal_aspects()
    end
end

def addPRLabel(label)
    pr_number = github.pr_json["number"].to_i

  if github.pr_labels.include?(label) == false
       github.api.add_labels_to_an_issue("TestAppCi", pr_number, [label])
  end
end

def add_internal_aspects()
    markdown("# :rotating_light: You are merging into **internal**. :rotating_light:")
    addPRLabel('for-regression')
end

def fail_and_notify(msg)
    fail(msg)
    pr_author = TEAM_MAPPING[github.pr_author]
    notify_slack(":rotating_light: @#{pr_author} <#{github.pr_json["html_url"]}|#{github.pr_title}> :rotating_light:")
end

def infer_check()
    value = system("infer --fail-on-issue -- ./gradlew clean :app:assembleDebug --stacktrace")
    if value
        message("Infer Check passed :zap:")
    else
        fail_and_notify("Infer failed :red_circle: . Review console logs for more details")
    end
end

def gradle_run()
    if !gradlew_exists?
        fail_and_notify("Could not find `gradlew` inside current directory")
        return
    end
 
    detektValue = system("./gradlew detekt --no-daemon")

    if detektValue
        message("detekt passed :cake:")
    else
        fail_and_notify("detekt failed :cry: Review console logs for more details.")
        return
    end

    value = system("./gradlew testDebug assembleDebug staticAnalysis --no-daemon")

    # Check detekt report file
    if (File.exists?(detekt_report_file))
        issues = read_issues_from_report(detekt_report_file)
        send_inline_comment(issues)
    end

    # Check findbugs report file
    if (File.exists?(findbugs_report_file))
       message("Findbugs Report:  `#{findbugs_report_file}`")
    end

    if value
        message("Compilation passed :elephant:")
    else
        fail_and_notify("Gradle Compilation failed :scream: . Review console logs for more details.")
    end
    value
end

def parse_permissions()
    Dir.chdir("script") do
        setup = system("pip2 install -r requirements.txt")
        if setup 
            value = `python2 parse_permissions.py ../app/build/outputs/apk/app-debug.apk`
            if value.to_s.empty?
                message("No extra permissions added! Great! :sweat_smile:")

                pr_author = TEAM_MAPPING[github.pr_author]
                notify_slack("@#{pr_author} <#{github.pr_json["html_url"]}|#{github.pr_title}>")
            else
                fail_and_notify(value)
            end
        end
    end
end

# generates danger report but formatted for slack when text is nil
def notify_slack(msg = nil)
    slack.notify(channel: '#android-github', text: msg)
end

def gradlew_exists?
      `ls gradlew`.strip.empty? == false
end

def read_issues_from_report(report_file)
 file = File.open(report_file)
 document = Oga.parse_xml(file)
 document.xpath('//file')
end

# Send inline comment with danger's warn or fail method
#
# @return [void]
def send_inline_comment (issues)
      target_files = (git.modified_files - git.deleted_files) + git.added_files
      dir = "#{Dir.pwd}/"
      issues.each do |issue|
      	filename = issue.get('name')
      	if (target_files.include? filename)
      	  error = issue.at_xpath('error')
      	  error_message = error.get('message')
      	  line = (error.get('line') || "0").to_i
          send("warn", error_message, file: filename, line: line)
          end
      end
end

if gradle_run()
    notify_slack()
    parse_permissions()
else 
    notify_slack()
end

pr_report()
