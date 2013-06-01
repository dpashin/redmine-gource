require 'mysql2'
require "digest/md5"
require "open-uri"
require File.expand_path('./', 'config.rb')

# extact data from redmine database
task :extract do
  project_tree = build_tree($connection, 'projects', 'name')
  users = build_users($connection)
  history = query_history($connection).to_a
  known_path = {}
  users[nil] = 'unassigned'

  known_assignee = build_first_known_assignee(history)

  events = scan_issues($connection, known_assignee, known_path)

  history.each do |row|
    if !row['new_assignee'].nil?
      assignee = row['new_assignee'].to_i
    elsif known_assignee.has_key? row['issue_id']
      assignee = known_assignee[row['issue_id']]
    else
      assignee = row['assigned_to_id']
    end
    known_assignee[row['issue_id']] = assignee

    path = create_path(row, assignee)

    is_closed = (row['new_status_id'] == '5')

    old_path = known_path.has_key?(row['issue_id']) ? known_path[row['issue_id']] : nil

    if path == old_path && is_closed
      events << create_event('D', row, path)
      known_path.delete row['issue_id']
    elsif path == old_path && !is_closed
      events << create_event('M', row, path)

    elsif path != old_path && is_closed
      events << create_event('D', row, old_path) unless old_path.nil?
      known_path.delete row['issue_id']

    elsif path != old_path && !is_closed
      events << create_event('D', row, old_path) unless old_path.nil?
      events << create_event('A', row, path)
      known_path[row['issue_id']] = path
    end
  end

  events.sort! { |a, b| a[:timestamp] <=> b[:timestamp] }
  events.each do |e|
    path = e[:path]
    user_name = users.include?(path[:assignee]) ? users[path[:assignee]] : 'unassigned'

    # user_name = user_name.gsub(/(.+\s.).+/, '\1.')

    full_path = project_tree[path[:project_id]] + [user_name] + [path[:issue_id]]
    row = [e[:timestamp], users[e[:user_id]], e[:state], full_path.join('/')]
    puts row.join('|')
  end
end

# download gravatars for all users
task :gravatars do
  size = 90
  output_dir = './gravatars'

  $connection.query("
    SELECT firstname, lastname, mail FROM users
  ").each do |row|
    url = [
        "http://www.gravatar.com/avatar/",
        Digest::MD5.hexdigest(row['mail'].downcase),
        "?d=404&size=",
        size
    ].join('')
    filename = output_dir + '/' + row['firstname'] + ' ' + row['lastname'] + '.png'

    puts filename + ' ' + url
    begin
      content = open(url).read
      File.open(filename, 'wb') do |f|
        f.write content
      end
    rescue OpenURI::HTTPError => ex
      puts 'err: ' + filename
    end
  end
end

# build tree: { id => [dir, subdir1, ...] }
def build_tree(connection, table_name, name_field, lookup_tree = nil, detail_key = nil)
  result = {}

  detail_key_clause = detail_key.nil? ? '' : ', ' + detail_key
  sql = "
    SELECT
      id, parent_id, lft,rgt, #{name_field} AS name #{detail_key_clause}
    FROM #{table_name}
    ORDER BY lft asc, rgt DESC
  "

  connection.query(sql).each do |row|
    if row['parent_id'].nil?
      path = lookup_tree.nil? ? [] : lookup_tree[row[detail_key]].dup
    else
      path = result[row['parent_id']].dup
    end
    path << row[name_field]
    result[row['id']] = path
  end

  return result
end

def build_users(connection)
  query = connection.query("
    SELECT
      id,
      CONCAT(firstname, ' ', lastname) as user_name
    FROM
      users
  ")
  result = {}
  query.each do |row|
    result[row['id']] = row['user_name']
  end
  return result
end

def query_history(connection)
  connection.query("
    SELECT
      UNIX_TIMESTAMP(j.created_on) as timestamp,
      u.id as user_id,
      i.author_id,
      i.assigned_to_id,
      i.subject as issue_subject,
      i.project_id,
      j.journalized_id as issue_id,
      j.id as journal_id,
      (
        SELECT MAX(value) FROM journal_details d WHERE d.journal_id = j.id
        AND d.prop_key = 'assigned_to_id'
      ) as new_assignee,
      (
        SELECT MAX(old_value) FROM journal_details d WHERE d.journal_id = j.id
        AND d.prop_key = 'assigned_to_id'
      ) as old_assignee,
      (
        SELECT MAX(value) FROM journal_details d WHERE d.journal_id = j.id
        AND d.prop_key = 'status_id'
      ) as new_status_id

    FROM
      journals j, issues i, users u
    WHERE
      j.journalized_type = 'Issue'
      AND i.id = j.journalized_id
      AND u.id = j.user_id
    ORDER BY j.created_on, j.id
  ")
end

def create_event(state, row, path)
  {
      timestamp: row['timestamp'],
      user_id: row['user_id'],
      state: state,
      path: path
  }
end


def build_first_known_assignee(history)
  result = {}
  history.reverse.each do |row|
    unless row['old_assignee'].nil?
      result[row['issue_id'].to_i] = row['old_assignee'].to_i
    end
  end
  return result
end

def scan_issues(connection, first_known_assignee, known_path)
  result = []

  connection.query("
    SELECT
      UNIX_TIMESTAMP(created_on) as timestamp,
      id AS issue_id,
      project_id,
      author_id AS user_id,
      created_on,
      assigned_to_id
    FROM issues
  ").each do |row|
    if first_known_assignee[row['issue_id']].nil?
      assignee = row['assigned_to_id']
      first_known_assignee[row['issue_id'].to_i] = assignee
    else
      assignee = first_known_assignee[row['issue_id']]
    end
    path = create_path(row, assignee)
    known_path[row['issue_id']] = path
    e = create_event('A', row, path)
    result << e
  end
  result
end

def create_path(row, assignee)
  {
      project_id: row['project_id'],
      issue_id: row['issue_id'],
      assignee: assignee
  }
end