# This file contains the fastlane.tools configuration
# You can find the documentation at https://docs.fastlane.tools
#
# For a list of all available actions, check out
#
#     https://docs.fastlane.tools/actions
#

# Uncomment the line if you want fastlane to automatically update itself
# update_fastlane

default_platform(:ios)

platform :ios do

  desc "Push a new release build to the App Store"
  lane :release do |options|
    tag = ENV['TRAVIS_TAG'] || ''

    if tag =~ /.*-android/
      next
    end

    target = ENV["CRU_TARGET"]

    automatic_release = options.key?(:auto_release) && options[:auto_release] || false
    include_metadata = options.key?(:include_metadata) && options[:include_metadata] || false
    submit_for_review = options.key?(:submit) ? options[:submit] : true
    
    version_number = get_version_number(
        target: target
    )

    build_number = get_build_number

    upload_to_app_store(
        api_key_path: ENV['CRU_API_KEY_PATH'],
        username: ENV['CRU_FASTLANE_USERNAME'],
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
        build_number: build_number,
        dev_portal_team_id: ENV["CRU_DEV_PORTAL_TEAM_ID"],
        skip_screenshots: true,
        skip_metadata: !include_metadata,
        app_version: version_number,
        automatic_release: automatic_release,
        submit_for_review: submit_for_review,
        submission_information: cru_submission_information
    )

    cru_bump_version_number(version_number: version_number)

    cru_notify_users(message: "#{target} iOS Release Build #{version_number} (#{build_number}) submitted to App Store.")
    if submit_for_review
      cru_notify_users(message: "Build has been submitted for review and will be #{automatic_release ? 'automatically' : 'manually'} released.")
    end

  end

  desc "Push a new (beta) release build to TestFlight"
  lane :beta do
    target = ENV["CRU_TARGET"]

    build_number = increment_build_number({
      build_number: latest_testflight_build_number({
        api_key_path: ENV['CRU_API_KEY_PATH'],
        username: ENV['CRU_FASTLANE_USERNAME'],
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
      }) + 1
    })
    version_number =  get_version_number(
        target: target
    )
    build_branch = ENV['TRAVIS_BRANCH']

    sh('git', 'checkout', build_branch)

    ipa_path = cru_build_app

    testflight(
        api_key_path: ENV['CRU_API_KEY_PATH'],
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
        ipa: ipa_path,
        dev_portal_team_id: ENV["CRU_DEV_PORTAL_TEAM_ID"],
        changelog: ENV["TRAVIS_COMMIT_MESSAGE"]
    )

    cru_notify_users(message: "#{target} iOS Beta Build ##{build_number} released to TestFlight.")
  end

  # Localization functions

  desc "Download latest localization files from Onesky"
  lane :cru_download_localizations do
    locales = ENV["ONESKY_ENABLED_LOCALIZATIONS"].split(',')
    filename = ENV["ONESKY_FILENAME"]
    scheme = ENV['CRU_SCHEME']

    locales.each do |locale|
      begin
        dir = "../#{scheme}/#{locale}.lproj"
        Dir.mkdir(dir) unless File.exists?(dir)
        onesky_download(
            public_key: ENV["ONESKY_PUBLIC_KEY"],
            secret_key: ENV["ONESKY_SECRET_KEY"],
            project_id: ENV["ONESKY_PROJECT_ID"],
            locale: locale,
            filename: filename,
            destination: "./#{scheme}/#{locale}.lproj/#{filename}"
        )
      rescue
        puts("Failed to import #{locale}")
      end
    end

    cru_commit_localization_files(filename: filename)
  end

  desc 'Commit downloaded localization files to default branch and push to remote'
  lane :cru_commit_localization_files do |options|
    filename = options[:filename]

    begin
      git_add(path: "*/#{filename}")
      git_commit(path: "*/#{filename}",
                 message: "[skip ci] Adding latest localization files from Onesky")
    rescue
      puts("Failed to commit localization files.. maybe none to commit?")
    end
  end

  # Helper functions
  
  lane :cru_build_app do |options|
    profile_name = options[:profile_name] || ENV["CRU_APPSTORE_PROFILE_NAME"]
    type = options[:type] || 'appstore'
    export_method = options[:export_method] || 'app-store'

    if ENV['CRU_SKIP_LOCALIZATION_DOWNLOAD'].nil?
      cru_download_localizations
    end

    automatic_code_signing(
        use_automatic_signing: false,
        profile_name: profile_name
    )


    unless options.key?(:skip_create_keychain) && options[:skip_create_keychain]
      # Travis requires a keychain to be created to store the certificates in, however
      # using this utility to create a keychain locally will really mess up local keychains
      # and is not required for a successful build.
      # It also cannot be called more than once (in the case that cru_build_app happens more than once in the same execution)
      create_keychain(
          name: ENV["MATCH_KEYCHAIN_NAME"],
          password: ENV["MATCH_PASSWORD"],
          default_keychain: true,
          unlock: true,
          timeout: 3600,
          add_to_search_list: true
        )
    end
    unless ENV["CRU_CALLDIRECTORY_TARGET"].nil?
      call_directory_profile  = type == "adhoc" ? ENV["CRU_CALLDIRECTORY_ADHOC_PROFILE_NAME"] : ENV["CRU_CALLDIRECTORY_APPSTORE_PROFILE_NAME"]
      automatic_code_signing(
          use_automatic_signing: false,
          targets: ENV["CRU_CALLDIRECTORY_TARGET"],
          profile_name: call_directory_profile
      )
    end

    cru_fetch_certs(type: type)

    if ENV["CRU_SKIP_COCOAPODS"].nil?
      cocoapods(
          podfile: './Podfile',
          try_repo_update_on_error: true
      )
    end

    gym(
    scheme: ENV["CRU_SCHEME"],
        export_method: export_method,
        export_options: {
            provisioningProfiles: {
                ENV["CRU_APP_IDENTIFIER"] => profile_name
            }
        }
    )
  end

  lane :cru_build_adhoc do |options|
    unless ENV["CRU_ADHOC_PROFILE_NAME"].nil?
      target = ENV["CRU_TARGET"]
      version_number =  get_version_number(
        target: target
      )

      github_ipa_release_path = cru_build_app(
                      profile_name: ENV["CRU_ADHOC_PROFILE_NAME"], 
                      type: "adhoc", 
                      export_method: "ad-hoc",
                      skip_create_keychain: options[:skip_create_keychain] || false)

      cru_push_release_to_github(
        version_number: version_number,
        project_name: target,
        ipa_path: github_ipa_release_path
      )

      push_to_git_remote
    end
  end

  lane :cru_fetch_certs do |options|
    match(type: options[:type],
      api_key_path: ENV['CRU_API_KEY_PATH'],
      username: ENV['CRU_FASTLANE_USERNAME'],
      app_identifier: ENV['CRU_APP_IDENTIFIER'],
      keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
      keychain_password: ENV["MATCH_PASSWORD"])

    unless ENV["CRU_CALLDIRECTORY_APP_IDENTIFIER"].nil?
      match(type: options[:type],
          api_key_path: ENV['CRU_API_KEY_PATH'],
          username: ENV['CRU_FASTLANE_USERNAME'],
          app_identifier: ENV['CRU_CALLDIRECTORY_APP_IDENTIFIER'],
          keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
          keychain_password: ENV["MATCH_PASSWORD"])
    end 
  end

  lane :cru_update_commit do |options|
    project_file_path = "#{ENV['CRU_XCODEPROJ']}/project.pbxproj"
    info_file_path = "#{ENV['CRU_SCHEME']}/Info.plist"
    git_commit(path:[project_file_path, info_file_path],
               message: options[:message])
  end

  lane :cru_bump_version_number do |params|
    if "v#{params[:version_number]}".eql? ENV['TRAVIS_TAG']

      sh('git remote set-branches --add origin master')
      sh('git fetch origin master:master')
      sh('git checkout master')
      version_number = increment_version_number(bump_type: 'patch')
      cru_update_commit(message: "[skip ci] Bumping version number to #{version_number} for next build")
      push_to_git_remote
    end
  end

  lane :cru_notify_users do |options|
    if ENV["SLACK_URL"]
      slack(
        slack_url: ENV["SLACK_URL"],
        fail_on_error: false,
        message: options[:message]
      )
    end
  end

  def cru_submission_information
    {
        export_compliance_available_on_french_store: false,
        export_compliance_contains_proprietary_cryptography: false,
        export_compliance_contains_third_party_cryptography: false,
        export_compliance_is_exempt: false,
        export_compliance_uses_encryption: false,
        export_compliance_app_type: nil,
        export_compliance_encryption_updated: false,
        export_compliance_compliance_required: false,
        export_compliance_platform: "ios",
        content_rights_contains_third_party_content: false,
        content_rights_has_rights: false,
        add_id_info_serves_ads: ENV['CRU_IDFA_SERVES_ADS'] || false,
        add_id_info_tracks_action: ENV['CRU_IDFA_TRACKS_ACTION'] || false,
        add_id_info_tracks_install: ENV['CRU_IDFA_TRACKS_INSTALL'] || false,
        add_id_info_uses_idfa: ENV['CRU_IDFA_IS_ENABLED'] || false
    }
  end

  lane :cru_push_release_to_github do |params|
    version = params[:version_number]
    project_name = params[:project_name]
    build = ENV["TRAVIS_BUILD_NUMBER"]
    ipa_path = params[:ipa_path]

    set_github_release(
        repository_name: ENV["TRAVIS_REPO_SLUG"],
        api_token: ENV["CI_USER_TOKEN"],
        name: "#{project_name} beta release  ##{version}-#{build}",
        tag_name: "#{version}-#{build}",
        description: "",
        commitish: ENV["TRAVIS_BRANCH"],
        upload_assets: [ipa_path],
        is_prerelease: true
    )
  end

  # Shared Lanes

  # Runs your Xcode tests for the provided scheme.
  #
  # fastlane action: https://docs.fastlane.tools/actions/run_tests/
  #
  # options:
  # - scheme: Specificy the name of the scheme you want to run tests on.  The scheme should be marked as shared in Xcode.
  #
  lane :cru_shared_lane_run_tests do |options|

    scheme = options[:scheme] || ENV["RUN_TESTS_SCHEME"]

    run_tests(
        scheme: scheme
    )
  end

  # Downloads OneSky localizations by locale to the Xcode project directory and then git adds and commits the localization file.
  #
  # plugin: https://github.com/thekie/fastlane-plugin-onesky
  #
  # options:
  # - one_sky_download_to_project_directory: The directory name localization files should be downloaded to.
  # - one_sky_filename:
  # - one_sky_localizations: Comma separated string list of locales to download from OneSky.
  # - one_sky_project_id:
  # - one_sky_public_key:
  # - one_sky_secret_key:
  #
  lane :cru_shared_lane_download_and_commit_latest_one_sky_localizations do |options|

    one_sky_download_to_project_directory = options[:one_sky_download_to_project_directory] || ENV["ONESKY_DOWNLOAD_TO_XCODE_PROJECT_DIRECTORY"]
    one_sky_filename = options[:one_sky_filename] || ENV["ONESKY_FILENAME"]
    one_sky_localizations_string = options[:one_sky_localizations] || ENV["ONESKY_LOCALIZATIONS"]
    one_sky_project_id = options[:one_sky_project_id] || ENV["ONESKY_PROJECT_ID"]
    one_sky_public_key = options[:one_sky_public_key] || ENV["ONESKY_PUBLIC_KEY"]
    one_sky_secret_key = options[:one_sky_secret_key] || ENV["ONESKY_SECRET_KEY"] || ""
    
    one_sky_localizations_array = one_sky_localizations_string.split(",")

    one_sky_localizations_array.each do |locale|
        
        begin
            
            directory = "../#{one_sky_download_to_project_directory}/#{locale}.lproj"
            Dir.mkdir(directory) unless File.exists?(directory)
            
            onesky_download(
                destination: "./#{one_sky_download_to_project_directory}/#{locale}.lproj/#{one_sky_filename}",
                filename: one_sky_filename,
                locale: locale,
                project_id: one_sky_project_id,
                public_key: one_sky_public_key,
                secret_key: one_sky_secret_key
            )
        rescue
            puts("Failed to import #{locale}")
        end
    end

    begin
        git_add(path: "*/#{one_sky_filename}")
        git_commit(
            path: "*/#{one_sky_filename}",
            message: "[skip ci] Adding latest localization files from OneSky"
        )
        push_to_git_remote
    rescue
      puts("Failed to commit localization files.. maybe none to commit?")
    end
  end

  # Increments the xcode project build number by 1 using the latest testflight build number to increment against.
  #
  # fastlane action: https://docs.fastlane.tools/actions/increment_build_number/
  # fastlane action: https://docs.fastlane.tools/actions/latest_testflight_build_number/
  #
  # options:
  # - api_key_path: Path to your App Store Connect API Key JSON file.
  # - app_release_bundle_identifier: The bundle identifier for your xcode release configuration.
  #
  lane :cru_shared_lane_increment_xcode_project_build_number do |options|

    api_key_path = options[:api_key_path] || ENV["APP_STORE_CONNECT_API_KEY_JSON_FILE_PATH"]
    app_release_bundle_identifier = options[:app_release_bundle_identifier] || ENV["APP_RELEASE_BUNDLE_IDENTIFIER"]

    latest_build_number = latest_testflight_build_number(
        api_key_path: api_key_path,
        app_identifier: app_release_bundle_identifier
    )

    specific_build_number = latest_build_number + 1

    increment_build_number(build_number: specific_build_number)
  end

  # First updates the xcode project code signing settings for each provided target, setting automatic code signing to false and setting the provisioning profile name.
  # Then uses match to fetch certificates from a git storage and adds them to the xcode project based on code signing settings.
  #
  # fastlane action: https://docs.fastlane.tools/actions/update_code_signing_settings/
  # fastlane action: https://docs.fastlane.tools/actions/create_keychain/
  # fastlane action: https://docs.fastlane.tools/actions/match/
  # fastlane action: https://docs.fastlane.tools/actions/gym/
  # fastlane action: https://docs.fastlane.tools/actions/testflight/
  #
  # options:
  # - api_key_path: Path to your App Store Connect API Key JSON file. Required for match.
  # - app_release_bundle_identifier: The bundle identifier for your xcode release configuration.
  # - code_signing_app_bundle_ids: Comma separated string of bundle ids that require code signing.  This comma separated list should match up with the code signing provisioning profile names and targets.
  # - code_signing_provisioning_profile_names: Comma separated string of provisioning profile names that require code signing.  This comma separated list should match up with the code signing app bundle ids and targets.
  # - code_signing_targets: Comma separated string of targets that require code signing.  This comma separated list should match up with the code signing app bundle ids and provisioning profile names.
  # - deploy_type: 
  # - is_running_in_ci: If running fastlane from CI, this is used to create a keychain needed for match.  Local fastlane builds should not need to set this.
  # - match_git_branch:
  # - match_git_url:
  # - match_keychain_name:
  #
  lane :cru_shared_lane_build_and_deploy_for_testflight_release do |options|

    # Update code signing settings

    api_key_path = options[:api_key_path] || ENV["APP_STORE_CONNECT_API_KEY_JSON_FILE_PATH"]
    app_release_bundle_identifier = options[:app_release_bundle_identifier] || ENV["APP_RELEASE_BUNDLE_IDENTIFIER"]
    code_signing_app_bundle_ids = options[:code_signing_app_bundle_ids] || ENV["CODE_SIGNING_APP_BUNDLE_IDS"]
    code_signing_provisioning_profile_names = options[:code_signing_provisioning_profile_names] || ENV["CODE_SIGNING_PROVISIONING_PROFILE_NAMES"]
    code_signing_targets = options[:code_signing_targets] || ENV["CODE_SIGNING_TARGETS"]
    is_running_in_ci = options[:is_running_in_ci] || false
    match_git_branch = options[:match_git_branch] || ENV["MATCH_GIT_BRANCH"]
    match_git_url = options[:match_git_url] || ENV["MATCH_GIT_URL"]
    match_keychain_name = options[:match_keychain_name] || ENV["MATCH_KEYCHAIN_NAME"]

    app_bundle_ids_array = code_signing_app_bundle_ids.split(",")
    profile_names_array = code_signing_provisioning_profile_names.split(",")
    targets_array = code_signing_targets.split(",")

    targets_array.each_with_index do |target, index|

        profile_name = profile_names_array[index]

        puts "update code signing for target: #{target}, profile_name: #{profile_name}"

        update_code_signing_settings(
            use_automatic_signing: false, 
            targets: target,
            profile_name: profile_name
        )
    end

    # Create Keychain needed for CI.

    if is_running_in_ci
        
        puts "creating keychain for ci..."
        
        create_keychain(
          name: match_keychain_name,
          password: ENV["MATCH_PASSWORD"],
          default_keychain: true,
          unlock: true,
          timeout: 3600,
          add_to_search_list: true
        )
    else

        puts "skipping create keychain..."
    end

    # Match - Release

    match(
        api_key_path: api_key_path,
        app_identifier: app_bundle_ids_array,
        git_basic_authorization: Base64.strict_encode64(ENV["MATCH_GIT_BASIC_AUTHORIZATION_PAT"]),
        git_branch: match_git_branch,
        git_url: match_git_url,
        keychain_name: match_keychain_name,
        keychain_password: ENV["MATCH_PASSWORD"],
        platform: "ios",
        storage_mode: "git",
        type: "appstore"
    )

    # Gym - Release

    release_ipa_path = gym(
        scheme: ENV["GYM_RELEASE_SCHEME"],
        export_method: "app-store",
        export_options: {
            provisioningProfiles: {
                ENV["GYM_RELEASE_APP_BUNDLE_IDENTIFIER"] => ENV["GYM_RELEASE_PROVISIONING_PROFILE"]
            }
        }
    )

    # TestFlight - Release

    testflight(
        api_key_path: api_key_path,
        app_identifier: app_release_bundle_identifier,
        ipa: release_ipa_path,
        skip_waiting_for_build_processing: true
    )
  end
end
